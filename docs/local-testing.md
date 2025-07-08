# Local Testing Configuration Guide

This document outlines the local testing setup for the Cloudflare Workers CI/CD platform, including act (local GitHub Actions) and Miniflare integration for sub-10s feedback loops.

## act - Local GitHub Actions Testing

### Installation

```bash
# Install act
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Or using Homebrew on macOS
brew install act

# Or using chocolatey on Windows
choco install act-cli

# Verify installation
act --version
```

### Configuration

Create `.actrc` in your project root:

```bash
# .actrc - act configuration
--platform ubuntu-latest=ghcr.io/catthehacker/ubuntu:act-latest
--platform ubuntu-22.04=ghcr.io/catthehacker/ubuntu:act-22.04
--platform ubuntu-20.04=ghcr.io/catthehacker/ubuntu:act-20.04

# Use smaller images for faster startup
--platform ubuntu-latest=ghcr.io/catthehacker/ubuntu:act-latest-micro

# Resource limits
--cpus 2
--memory 4g

# Environment variables
--env GITHUB_TOKEN=dummy_token
--env NODE_ENV=test
```

### Environment Variables for Testing

Create `.env` file for local testing:

```bash
# .env - Local environment variables
CLOUDFLARE_API_TOKEN=dummy_token_for_testing
DATADOG_API_KEY=dummy_key_for_testing
DATADOG_APP_KEY=dummy_app_key_for_testing
WORKER_NAME=test-worker
ENVIRONMENT=test
```

### Act Workflow Configuration

Create `.github/workflows/act-test.yml` for local testing:

```yaml
name: Local Testing with act

on:
  workflow_dispatch:
  push:
    branches: [local-test]

jobs:
  local-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run local tests
        run: npm test
      
      - name: Validate wrangler config
        run: npx wrangler validate
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN || env.CLOUDFLARE_API_TOKEN }}
      
      - name: Build worker
        run: |
          if npm run build --if-present; then
            echo "Build completed"
          else
            echo "No build script found"
          fi
      
      - name: Test local development
        run: |
          timeout 10s npm run dev > dev.log 2>&1 &
          sleep 5
          curl -f http://localhost:8787/ || echo "Dev server test failed"
          pkill -f "npm run dev" || true
        continue-on-error: true
```

### Running Local Tests

```bash
# Test specific workflow
act -j local-ci

# Test with specific event
act push -e .github/workflows/test-event.json

# Test reusable workflow
act workflow_call -W .github/workflows/ci.yml

# Test with secrets
act -s CLOUDFLARE_API_TOKEN=dummy_token

# Test specific job
act -j test

# Dry run (show what would be executed)
act --dryrun

# Verbose output
act -v

# Use specific platform
act --platform ubuntu-latest=node:18
```

### Custom Event Payloads

Create `.github/workflows/test-event.json`:

```json
{
  "push": {
    "ref": "refs/heads/main",
    "repository": {
      "name": "test-repo",
      "full_name": "user/test-repo"
    },
    "head_commit": {
      "id": "abc123",
      "message": "Test commit"
    }
  }
}
```

### act Performance Optimization

```bash
# .actrc optimizations
--pull=false  # Don't pull images every time
--reuse       # Reuse containers
--rm=false    # Don't remove containers (for debugging)

# Use act cache
--cache-dir ~/.cache/act

# Bind mount node_modules for faster installs
--bind /host/path/node_modules:/workspace/node_modules
```

## Miniflare - Local Workers Runtime

### Installation and Setup

```bash
# Install Miniflare
npm install -g miniflare

# Or as dev dependency
npm install --save-dev miniflare
```

### Miniflare Configuration

Create `miniflare.config.js`:

```javascript
// miniflare.config.js
export default {
  // Script configuration
  script: "./src/index.js",
  modules: true,
  
  // Runtime configuration
  compatibilityDate: "2023-05-18",
  compatibilityFlags: ["nodejs_compat"],
  
  // Development options
  port: 8787,
  host: "localhost",
  verbose: true,
  
  // Storage
  persist: "./.miniflare-storage",
  
  // KV Namespaces
  kvNamespaces: {
    "MY_KV": "kv_namespace_1"
  },
  
  // Durable Objects
  durableObjects: {
    "MY_DO": "MyDurableObject"
  },
  
  // R2 Buckets
  r2Buckets: {
    "MY_BUCKET": "test_bucket"
  },
  
  // Environment variables
  vars: {
    ENVIRONMENT: "development",
    API_VERSION: "v1"
  },
  
  // Secrets (for testing)
  secrets: {
    "SECRET_KEY": "test_secret"
  },
  
  // Bindings for testing
  bindings: {
    "DATABASE_URL": "sqlite://test.db"
  }
};
```

### Vitest Integration with Miniflare

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    pool: '@cloudflare/vitest-pool-workers',
    poolOptions: {
      workers: {
        wrangler: {
          configPath: './wrangler.toml',
        },
        miniflare: {
          // Miniflare options
          compatibilityDate: '2023-05-18',
          compatibilityFlags: ['nodejs_compat'],
          
          // Bindings for testing
          bindings: {
            ENVIRONMENT: 'test',
            TEST_MODE: 'true'
          },
          
          // KV for testing
          kvNamespaces: {
            TEST_KV: 'test_namespace'
          },
          
          // Persistence for integration tests
          persist: './.miniflare-test-storage',
          
          // Custom modules
          modules: [
            {
              type: 'ESModule',
              path: './src/index.js'
            }
          ]
        }
      }
    },
    environment: 'miniflare',
    globals: true,
    testTimeout: 10000
  }
});
```

### Fast Test Suite Example

```javascript
// test/fast.test.js - Sub-10s test suite
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { unstable_dev } from 'wrangler';

describe('Fast Worker Tests', () => {
  let worker;

  beforeAll(async () => {
    // Start worker in test mode (should be <2s)
    worker = await unstable_dev('src/index.js', {
      experimental: { disableExperimentalWarning: true },
      vars: { ENVIRONMENT: 'test' }
    });
  });

  afterAll(async () => {
    await worker.stop();
  });

  it('should respond to health check quickly', async () => {
    const start = Date.now();
    const resp = await worker.fetch('/health');
    const duration = Date.now() - start;
    
    expect(resp.status).toBe(200);
    expect(duration).toBeLessThan(100); // <100ms response
  });

  it('should handle API requests', async () => {
    const resp = await worker.fetch('/api/hello?name=Test');
    const data = await resp.json();
    
    expect(data.message).toBe('Hello, Test!');
    expect(data.environment).toBe('test');
  });

  it('should handle errors gracefully', async () => {
    const resp = await worker.fetch('/api/error');
    expect(resp.status).toBe(500);
  });
});
```

### Performance Test Suite

```javascript
// test/performance.test.js
import { describe, it, expect } from 'vitest';

describe('Performance Tests', () => {
  it('should start worker in <5 seconds', async () => {
    const start = Date.now();
    
    const worker = await unstable_dev('src/index.js', {
      experimental: { disableExperimentalWarning: true }
    });
    
    const startupTime = Date.now() - start;
    
    await worker.stop();
    
    expect(startupTime).toBeLessThan(5000);
  });

  it('should handle 100 concurrent requests', async () => {
    const worker = await unstable_dev('src/index.js');
    
    const requests = Array(100).fill().map(() => 
      worker.fetch('/api/hello')
    );
    
    const start = Date.now();
    const responses = await Promise.all(requests);
    const duration = Date.now() - start;
    
    expect(responses.every(r => r.status === 200)).toBe(true);
    expect(duration).toBeLessThan(2000); // <2s for 100 requests
    
    await worker.stop();
  });
});
```

## Integrated Development Workflow

### NPM Scripts for Fast Development

```json
{
  "scripts": {
    "dev": "miniflare --config miniflare.config.js",
    "dev:watch": "miniflare --config miniflare.config.js --watch",
    "test:fast": "vitest run --config vitest.config.js",
    "test:watch": "vitest --config vitest.config.js",
    "test:local": "act -j local-ci",
    "test:full": "npm run test:fast && npm run test:local",
    "validate:local": "wrangler validate && act --dryrun",
    "debug": "miniflare --config miniflare.config.js --debug --inspector-port 9229"
  }
}
```

### Pre-commit Hooks for Fast Feedback

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: fast-tests
        name: Fast test suite
        entry: npm run test:fast
        language: system
        pass_filenames: false
        
      - id: validate-config
        name: Validate wrangler config
        entry: wrangler validate
        language: system
        files: 'wrangler\.toml$'
        
      - id: lint
        name: Lint code
        entry: npm run lint
        language: system
        files: '\.(js|ts)$'
```

### VS Code Integration

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Start dev server",
      "type": "shell",
      "command": "npm run dev",
      "group": "build",
      "presentation": {
        "echo": true,
        "reveal": "always",
        "focus": false,
        "panel": "new"
      },
      "problemMatcher": []
    },
    {
      "label": "Run fast tests",
      "type": "shell",
      "command": "npm run test:fast",
      "group": "test",
      "presentation": {
        "echo": true,
        "reveal": "always",
        "focus": false,
        "panel": "new"
      }
    },
    {
      "label": "Test with act",
      "type": "shell",
      "command": "npm run test:local",
      "group": "test"
    }
  ]
}
```

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Worker",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "restart": true,
      "preLaunchTask": "Start dev server"
    }
  ]
}
```

## CI/CD Integration Testing

### GitHub Actions with act

```yaml
# .github/workflows/local-validation.yml
name: Local Validation

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  validate-local-testing:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup act
        run: |
          curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
          act --version
      
      - name: Test local CI workflow
        run: |
          # Test that act can run our workflows
          act -j local-ci --dryrun
      
      - name: Validate act configuration
        run: |
          # Verify .actrc exists and is valid
          test -f .actrc
          
          # Test that workflows can be parsed
          act --list
```

### Performance Benchmarking

```bash
#!/bin/bash
# scripts/benchmark-local.sh

echo "🚀 Starting local development benchmark..."

# Benchmark test startup time
echo "⏱️  Testing startup time..."
START_TIME=$(date +%s%N)
timeout 30s npm run test:fast
END_TIME=$(date +%s%N)
TEST_DURATION=$(( (END_TIME - START_TIME) / 1000000 ))

echo "Test duration: ${TEST_DURATION}ms"

if [ $TEST_DURATION -lt 10000 ]; then
  echo "✅ Tests completed in <10s (target met)"
else
  echo "❌ Tests took >10s (target missed)"
  exit 1
fi

# Benchmark dev server startup
echo "⏱️  Testing dev server startup..."
START_TIME=$(date +%s%N)
timeout 15s npm run dev > /dev/null 2>&1 &
DEV_PID=$!

sleep 2
if curl -f http://localhost:8787/health > /dev/null 2>&1; then
  END_TIME=$(date +%s%N)
  DEV_DURATION=$(( (END_TIME - START_TIME) / 1000000 ))
  echo "Dev server startup: ${DEV_DURATION}ms"
  
  if [ $DEV_DURATION -lt 5000 ]; then
    echo "✅ Dev server started in <5s (target met)"
  else
    echo "❌ Dev server took >5s (target missed)"
  fi
else
  echo "❌ Dev server failed to start"
fi

kill $DEV_PID 2>/dev/null || true

echo "🎉 Benchmark completed"
```

## Troubleshooting

### Common act Issues

```bash
# Issue: Permission denied
sudo chmod +x $(which act)

# Issue: Docker not found
sudo systemctl start docker

# Issue: Out of memory
act --platform ubuntu-latest=node:18-alpine

# Issue: Slow performance
act --reuse --pull=false

# Issue: Missing secrets
act -s GITHUB_TOKEN=dummy -s CLOUDFLARE_API_TOKEN=dummy
```

### Common Miniflare Issues

```bash
# Issue: Port already in use
lsof -ti:8787 | xargs kill -9

# Issue: Module not found
npm install --save-dev @cloudflare/workers-types

# Issue: Permission errors
rm -rf .miniflare-storage && mkdir .miniflare-storage

# Issue: Compatibility flags
miniflare --compatibility-date 2023-05-18 --compatibility-flags nodejs_compat
```

### Debug Configuration

```javascript
// debug.config.js
export default {
  script: "./src/index.js",
  modules: true,
  port: 8787,
  verbose: true,
  debug: true,
  inspectorPort: 9229,
  
  // Enhanced logging
  log: console.log,
  
  // Custom error handling
  handleRuntimeStdout: (data) => {
    console.log('[WORKER]', data.toString());
  },
  
  handleRuntimeStderr: (data) => {
    console.error('[WORKER ERROR]', data.toString());
  }
};
```

## Performance Targets

### Target Metrics

- **Test execution**: <10 seconds
- **Dev server startup**: <5 seconds  
- **act workflow validation**: <30 seconds
- **Miniflare worker startup**: <2 seconds
- **Full local CI cycle**: <60 seconds

### Monitoring Script

```bash
#!/bin/bash
# scripts/monitor-performance.sh

TARGET_TEST_TIME=10000  # 10 seconds
TARGET_DEV_TIME=5000    # 5 seconds

echo "📊 Performance Monitoring Report"
echo "================================"

# Test performance
START=$(date +%s%N)
npm run test:fast >/dev/null 2>&1
END=$(date +%s%N)
TEST_TIME=$(( (END - START) / 1000000 ))

echo "Test execution: ${TEST_TIME}ms (target: <${TARGET_TEST_TIME}ms)"

if [ $TEST_TIME -lt $TARGET_TEST_TIME ]; then
  echo "✅ Test performance: PASS"
else
  echo "❌ Test performance: FAIL"
fi

# Dev server performance
START=$(date +%s%N)
timeout 10s npm run dev >/dev/null 2>&1 &
DEV_PID=$!
sleep 2

if curl -f http://localhost:8787/health >/dev/null 2>&1; then
  END=$(date +%s%N)
  DEV_TIME=$(( (END - START) / 1000000 ))
  echo "Dev startup: ${DEV_TIME}ms (target: <${TARGET_DEV_TIME}ms)"
  
  if [ $DEV_TIME -lt $TARGET_DEV_TIME ]; then
    echo "✅ Dev performance: PASS"
  else
    echo "❌ Dev performance: FAIL"
  fi
else
  echo "❌ Dev server: FAILED TO START"
fi

kill $DEV_PID 2>/dev/null || true

echo "================================"
```

## Resources

- [act Documentation](https://github.com/nektos/act)
- [Miniflare Documentation](https://miniflare.dev/)
- [Vitest Workers Pool](https://github.com/cloudflare/workers-sdk/tree/main/packages/vitest-pool-workers)
- [Wrangler Testing](https://developers.cloudflare.com/workers/wrangler/testing/)

---

**Key Principle**: Local testing should provide immediate feedback (<10s) while maintaining high fidelity to production environments.