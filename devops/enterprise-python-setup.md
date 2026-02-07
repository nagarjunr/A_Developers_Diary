# Enterprise Python Setup: Corporate Networks, Proxies, and CA Certificates

A complete guide to setting up Python development environments in enterprise/corporate networks with SSL certificates, proxy configurations, and secure package installation.

## Table of Contents

1. [The Corporate Network Challenge](#the-corporate-network-challenge)
2. [Understanding the Problem](#understanding-the-problem)
3. [CA Certificate Management](#ca-certificate-management)
4. [Proxy Configuration](#proxy-configuration)
5. [Virtual Environment Setup](#virtual-environment-setup)
6. [Package Installation](#package-installation)
7. [Complete Setup Script](#complete-setup-script)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)

---

## The Corporate Network Challenge

**Common scenario:** You're developing Python applications in a corporate environment and encounter:

```
❌ SSL: CERTIFICATE_VERIFY_FAILED
❌ Connection refused: pypi.org
❌ Timeout when installing packages
❌ Git clone fails with SSL error
```

**Why this happens:**
- Corporate networks use **proxy servers** for internet access
- Proxy servers use **custom SSL certificates** (man-in-the-middle inspection)
- Python doesn't trust these certificates by default
- Package managers (pip) fail SSL verification

**This guide solves:**
✅ SSL certificate verification errors
✅ Proxy configuration for pip, requests, and curl
✅ Virtual environment isolation
✅ Secure package installation in corporate environments

---

## Understanding the Problem

### Normal SSL Flow (Home/Personal)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Your Computer]:::green --> B[Internet]:::blue
    B --> C[pypi.org]:::teal
    B -.-|✅ Trust PyPI's SSL| C

    classDef green fill:#50C878,stroke:#2E8B57,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
```

### Corporate SSL Flow (Enterprise)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#1f77b4','primaryTextColor':'#fff','primaryBorderColor':'#145a8c','lineColor':'#5a5a5a','secondaryColor':'#ff7f0e','tertiaryColor':'#2ca02c'}}}%%
graph LR
    A[Your Computer]:::orange --> B[Corporate Proxy]:::red
    B --> C[Internet]:::blue
    C --> D[pypi.org]:::teal
    B -.->|Custom CA Certificate<br/>Intercepts SSL| E[Certificate Store]:::purple

    classDef orange fill:#F39C12,stroke:#BA7A0A,color:#fff
    classDef red fill:#E74C3C,stroke:#A93226,color:#fff
    classDef blue fill:#4A90E2,stroke:#2E5C8A,color:#fff
    classDef teal fill:#1ABC9C,stroke:#138D75,color:#fff
    classDef purple fill:#9B59B6,stroke:#6C3483,color:#fff
```

**Problem:** Python doesn't trust the corporate proxy's SSL certificate.

**Solution:** Add corporate CA certificates to Python's trust store.

---

## CA Certificate Management

### Step 1: Locate Corporate Certificates

Corporate IT usually provides CA certificates as `.pem`, `.crt`, or `.cer` files.

**Common locations:**
- Email from IT department
- Company intranet download page
- Pre-installed on company laptops: `/usr/local/share/ca-certificates/` (Linux)
- Windows Certificate Store (export to file)

**Example structure:**
```
~/Downloads/corporate_certs/
├── root_ca.pem
├── intermediate_ca.pem
└── proxy_ca.crt
```

### Step 2: Merge Certificates into Single Bundle

Create a combined CA bundle file:

```bash
#!/bin/bash
# merge_ca_certs.sh

CA_SOURCE_DIR="$HOME/Downloads/corporate_certs"
OUTPUT_BUNDLE="./certs/enterprise_ca.pem"

mkdir -p ./certs
rm -f "$OUTPUT_BUNDLE"

# Find and concatenate all certificate files
find "$CA_SOURCE_DIR" -type f \( -name "*.pem" -o -name "*.crt" -o -name "*.cer" \) | while read -r cert; do
    cat "$cert" >> "$OUTPUT_BUNDLE"
    echo "" >> "$OUTPUT_BUNDLE"  # Add newline between certs
done

echo "Created enterprise CA bundle: $OUTPUT_BUNDLE"
```

**Run:**

```bash
chmod +x merge_ca_certs.sh
./merge_ca_certs.sh
```

### Step 3: Append to Python's Certifi Bundle

Python's `requests` library uses `certifi` for SSL verification:

```python
# append_to_certifi.py
import certifi
import os

# Path to certifi's CA bundle
certifi_bundle = certifi.where()

# Path to your enterprise CA bundle
enterprise_ca = "./certs/enterprise_ca.pem"

if os.path.exists(enterprise_ca):
    with open(certifi_bundle, 'ab') as out:
        with open(enterprise_ca, 'rb') as inp:
            out.write(b"\n")
            out.write(inp.read())
    print(f"Appended enterprise CAs to: {certifi_bundle}")
else:
    print(f"Enterprise CA bundle not found: {enterprise_ca}")
```

**Run:**

```bash
python append_to_certifi.py
```

### Step 4: Set Environment Variables

Export SSL certificate paths:

```bash
# Point Python, requests, and curl to the certifi bundle
export SSL_CERT_FILE=$(python -c 'import certifi; print(certifi.where())')
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
export CURL_CA_BUNDLE="$SSL_CERT_FILE"
```

**Add to your shell profile** (`~/.bashrc`, `~/.zshrc`):

```bash
# Corporate SSL certificates
export SSL_CERT_FILE=$(python3 -c 'import certifi; print(certifi.where())' 2>/dev/null)
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
export CURL_CA_BUNDLE="$SSL_CERT_FILE"
```

---

## Proxy Configuration

### Step 1: Get Proxy Information

Ask your IT department or check system proxy settings.

**Common formats:**
- `http://proxy.company.com:8080`
- `http://username:password@proxy.company.com:8080`

**Check current proxy (macOS):**

```bash
scutil --proxy | grep -i http
```

**Check current proxy (Linux):**

```bash
env | grep -i proxy
```

### Step 2: Set Proxy Environment Variables

```bash
# HTTP and HTTPS proxy
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"

# Lowercase versions (some tools check these)
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"

# Bypass proxy for these hosts
export NO_PROXY="localhost,127.0.0.1,.internal.company.com"
export no_proxy="$NO_PROXY"
```

### Step 3: Configure Proxy Exceptions (NO_PROXY)

Add hosts that should bypass the proxy:

```bash
export NO_PROXY="localhost,127.0.0.1,*.company.com,10.0.0.0/8"
```

**Why this matters:**
- Local development servers (localhost, 127.0.0.1)
- Internal company services (*.company.com)
- Container networks (Docker services)
- Cloud services (*.azure.com, *.openai.com for Azure OpenAI)

### Step 4: Add to Shell Profile

**File: `~/.bashrc` or `~/.zshrc`**

```bash
# Corporate proxy configuration
export HTTP_PROXY="http://proxy.company.com:8080"
export HTTPS_PROXY="http://proxy.company.com:8080"
export NO_PROXY="localhost,127.0.0.1,*.company.com"

# Lowercase versions
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export no_proxy="$NO_PROXY"
```

### Step 5: Configure Git for Proxy

```bash
# Set git proxy
git config --global http.proxy $HTTP_PROXY
git config --global https.proxy $HTTPS_PROXY

# Or with credentials
git config --global http.proxy http://username:password@proxy.company.com:8080

# Verify
git config --global --list | grep proxy
```

---

## Virtual Environment Setup

### Step 1: Choose Python Version

Check available Python versions:

```bash
# Check default
python3 --version

# Check specific versions
python3.11 --version
python3.12 --version

# Find all Python installations
ls -l /usr/bin/python* /usr/local/bin/python*
```

**Recommended:** Python 3.11+ for modern features and performance.

### Step 2: Create Virtual Environment

```bash
# Use specific Python version
python3.11 -m venv venv

# Or use latest available
python3 -m venv venv

# Activate (Unix/macOS)
source venv/bin/activate

# Activate (Windows)
venv\Scripts\activate
```

**Verify:**

```bash
# Should show path inside venv
which python
# Output: /path/to/your/project/venv/bin/python

# Check version
python --version
```

### Step 3: Upgrade pip

```bash
# Upgrade pip to latest version
python -m pip install --upgrade pip

# Verify
pip --version
```

---

## Package Installation

### Step 1: Install Certifi First

```bash
# Install certifi with trusted hosts (bypass SSL initially)
pip install --upgrade certifi \
    --trusted-host pypi.org \
    --trusted-host files.pythonhosted.org
```

**Why `--trusted-host`?**
- Temporarily bypasses SSL verification
- Allows initial installation before certificates are configured
- Only needed for the first certifi installation

### Step 2: Append Corporate CAs to Certifi

Run the Python script from earlier:

```bash
python - <<'EOF'
import certifi
import os

certifi_bundle = certifi.where()
enterprise_ca = "./certs/enterprise_ca.pem"

if os.path.exists(enterprise_ca) and os.path.getsize(enterprise_ca) > 0:
    with open(certifi_bundle, 'ab') as out:
        with open(enterprise_ca, 'rb') as inp:
            out.write(b"\n")
            out.write(inp.read())
    print(f"✅ Appended enterprise CAs to: {certifi_bundle}")
else:
    print("⚠️  Enterprise CA bundle not found or empty")
EOF
```

### Step 3: Install Requirements

Now pip will use the updated certifi bundle:

```bash
pip install -r requirements.txt
```

**If you still get SSL errors:**

```bash
# Use proxy explicitly
pip install -r requirements.txt --proxy $HTTP_PROXY

# Or use trusted hosts (less secure)
pip install -r requirements.txt \
    --trusted-host pypi.org \
    --trusted-host files.pythonhosted.org
```

---

## Complete Setup Script

**File: `setup.sh`**

```bash
#!/bin/bash
# Enterprise Python setup script
# Handles venv, CA certs, proxies, and dependencies

set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_ROOT"

echo "=== Enterprise Python Setup ==="

# 1. Choose Python version
PYTHON_BIN="python3"
if command -v python3.12 >/dev/null 2>&1; then
    PYTHON_BIN="python3.12"
elif command -v python3.11 >/dev/null 2>&1; then
    PYTHON_BIN="python3.11"
fi

echo "[1/6] Using Python: $PYTHON_BIN ($($PYTHON_BIN --version))"

# 2. Create virtual environment
VENV_DIR="venv"
if [ ! -d "$VENV_DIR" ]; then
    echo "[2/6] Creating virtual environment..."
    "$PYTHON_BIN" -m venv "$VENV_DIR"
else
    echo "[2/6] Virtual environment already exists"
fi

# Activate venv
source "$VENV_DIR/bin/activate"

# 3. Merge corporate CA certificates
echo "[3/6] Configuring SSL certificates..."
CERTS_DIR="./certs"
mkdir -p "$CERTS_DIR"
CA_SOURCE_DIR="${CA_CERTS_DIR:-$HOME/Downloads/corporate_certs}"
ENTERPRISE_CA="$CERTS_DIR/enterprise_ca.pem"

rm -f "$ENTERPRISE_CA"
if [ -d "$CA_SOURCE_DIR" ]; then
    echo "    - Building enterprise CA bundle from $CA_SOURCE_DIR"
    find "$CA_SOURCE_DIR" -type f \( -name "*.pem" -o -name "*.crt" -o -name "*.cer" \) | while read -r cert; do
        cat "$cert" >> "$ENTERPRISE_CA"
        echo "" >> "$ENTERPRISE_CA"
    done
    echo "    - Created: $ENTERPRISE_CA"
else
    echo "    ⚠️  CA source dir not found: $CA_SOURCE_DIR"
    echo "    ℹ️  Set CA_CERTS_DIR environment variable or place certs in ~/Downloads/corporate_certs"
    touch "$ENTERPRISE_CA"
fi

# 4. Install certifi and append enterprise CAs
echo "[4/6] Installing certifi..."
PIP_TRUSTED_HOSTS=("--trusted-host" "pypi.org" "--trusted-host" "files.pythonhosted.org")
python -m pip install --upgrade pip certifi "${PIP_TRUSTED_HOSTS[@]}" >/dev/null 2>&1

echo "[4/6] Appending enterprise CAs to certifi bundle..."
python - <<'PY'
import certifi, os
certifi_bundle = certifi.where()
enterprise_ca = os.path.join(os.getcwd(), 'certs', 'enterprise_ca.pem')
try:
    if os.path.exists(enterprise_ca) and os.path.getsize(enterprise_ca) > 0:
        with open(certifi_bundle, 'ab') as out, open(enterprise_ca, 'rb') as inp:
            out.write(b"\n")
            out.write(inp.read())
        print(f"    ✅ Appended to: {certifi_bundle}")
    else:
        print("    ⚠️  Enterprise CA bundle is empty")
except Exception as e:
    print(f"    ❌ Failed: {e}")
PY

# 5. Set environment variables
echo "[5/6] Configuring environment..."
export SSL_CERT_FILE="$(python -c 'import certifi; print(certifi.where())' 2>/dev/null)"
export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"
export CURL_CA_BUNDLE="$SSL_CERT_FILE"

# Load .env if exists
if [ -f .env ]; then
    echo "    - Loading .env file"
    set -a
    source .env
    set +a
fi

# Normalize proxy variables
export HTTP_PROXY="${HTTP_PROXY:-${http_proxy:-}}"
export HTTPS_PROXY="${HTTPS_PROXY:-${https_proxy:-}}"
export NO_PROXY="${NO_PROXY:-${no_proxy:-localhost,127.0.0.1}}"

echo "    - SSL_CERT_FILE: $SSL_CERT_FILE"
echo "    - HTTP_PROXY: ${HTTP_PROXY:-<not set>}"
echo "    - NO_PROXY: $NO_PROXY"

# 6. Install dependencies
if [ -f requirements.txt ]; then
    echo "[6/6] Installing dependencies..."
    python -m pip install -r requirements.txt --quiet
    echo "    ✅ Dependencies installed"
else
    echo "[6/6] No requirements.txt found, skipping"
fi

echo ""
echo "✅ Setup complete!"
echo ""
echo "Next steps:"
echo "  1. Activate the virtual environment:"
echo "     source venv/bin/activate"
echo ""
echo "  2. Run your application:"
echo "     python src/main.py"
echo ""
```

**Make it executable:**

```bash
chmod +x setup.sh
```

**Run:**

```bash
./setup.sh
```

---

## Troubleshooting

### Issue 1: SSL Certificate Verify Failed

**Error:**

```
ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED]
```

**Solutions:**

1. **Check certifi bundle:**
   ```bash
   python -c "import certifi; print(certifi.where())"
   # Verify enterprise CAs are appended (file should be larger)
   ls -lh $(python -c "import certifi; print(certifi.where())")
   ```

2. **Re-run certificate append:**
   ```bash
   python append_to_certifi.py
   ```

3. **Verify SSL environment variables:**
   ```bash
   echo $SSL_CERT_FILE
   echo $REQUESTS_CA_BUNDLE
   ```

4. **Test with requests:**
   ```python
   import requests
   response = requests.get("https://pypi.org")
   print(response.status_code)  # Should print 200
   ```

### Issue 2: Proxy Connection Failed

**Error:**

```
ProxyError: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded
```

**Solutions:**

1. **Verify proxy is set:**
   ```bash
   echo $HTTP_PROXY
   echo $HTTPS_PROXY
   ```

2. **Test proxy with curl:**
   ```bash
   curl -I https://pypi.org
   # Should return 200 OK
   ```

3. **Set proxy explicitly for pip:**
   ```bash
   pip install --proxy $HTTP_PROXY package-name
   ```

4. **Check NO_PROXY:**
   ```bash
   # Should include localhost
   echo $NO_PROXY
   ```

### Issue 3: Virtual Environment Not Activating

**Problem:** Commands still use system Python.

**Solutions:**

1. **Verify activation:**
   ```bash
   which python
   # Should show: /path/to/project/venv/bin/python
   ```

2. **Re-activate:**
   ```bash
   deactivate  # If already activated
   source venv/bin/activate
   ```

3. **Check PATH:**
   ```bash
   echo $PATH
   # Should start with: /path/to/project/venv/bin:...
   ```

### Issue 4: Pip Install Hangs

**Problem:** Pip command freezes during installation.

**Solutions:**

1. **Check network connectivity:**
   ```bash
   ping pypi.org
   curl -I https://pypi.org
   ```

2. **Increase timeout:**
   ```bash
   pip install --timeout 300 package-name
   ```

3. **Use verbose mode:**
   ```bash
   pip install -v package-name
   ```

4. **Clear pip cache:**
   ```bash
   pip cache purge
   ```

### Issue 5: Corporate Proxy Authentication Required

**Error:**

```
407 Proxy Authentication Required
```

**Solutions:**

1. **Add credentials to proxy URL:**
   ```bash
   export HTTP_PROXY="http://username:password@proxy.company.com:8080"
   ```

2. **URL-encode special characters:**
   ```bash
   # If password has special characters like @, !, encode them
   # @ → %40
   # ! → %21
   export HTTP_PROXY="http://user:p%40ssw%21rd@proxy.company.com:8080"
   ```

3. **Use NTLM authentication (Windows):**
   ```bash
   pip install --proxy http://DOMAIN\\username:password@proxy.company.com:8080 package-name
   ```

---

## Best Practices

### 1. Automate Setup with Script

Create a `setup.sh` script (like above) that:
- ✅ Creates venv
- ✅ Merges CA certificates
- ✅ Configures environment variables
- ✅ Installs dependencies

### 2. Document in README

**README.md:**

```markdown
## Enterprise Setup

This project requires additional configuration for corporate networks.

### Prerequisites
- Python 3.11+
- Corporate CA certificates (contact IT)
- Proxy configuration (check system settings)

### Setup

1. Place CA certificates in `~/Downloads/corporate_certs/`
2. Run setup script:
   ```bash
   ./setup.sh
   ```
3. Activate virtual environment:
   ```bash
   source venv/bin/activate
   ```

### Troubleshooting
See [docs/ENTERPRISE_SETUP.md](docs/ENTERPRISE_SETUP.md)
```

### 3. Use Environment Files

**.env:**

```bash
# Proxy configuration
HTTP_PROXY=http://proxy.company.com:8080
HTTPS_PROXY=http://proxy.company.com:8080
NO_PROXY=localhost,127.0.0.1,*.company.com

# CA certificate directory
CA_CERTS_DIR=/path/to/corporate/certs
```

**Load in script:**

```bash
if [ -f .env ]; then
    set -a
    source .env
    set +a
fi
```

### 4. Add to .gitignore

```
# .gitignore
venv/
.venv/
certs/
.env
*.pem
*.crt
```

**Never commit:**
- Virtual environments
- CA certificates
- Proxy credentials
- API keys

### 5. Test Installation

**test_setup.py:**

```python
#!/usr/bin/env python3
"""Test enterprise setup."""

import sys

def test_ssl():
    """Test SSL certificate configuration."""
    try:
        import requests
        response = requests.get("https://pypi.org", timeout=5)
        assert response.status_code == 200
        print("✅ SSL configuration: OK")
        return True
    except Exception as e:
        print(f"❌ SSL configuration failed: {e}")
        return False

def test_proxy():
    """Test proxy configuration."""
    import os
    if os.getenv("HTTP_PROXY") or os.getenv("HTTPS_PROXY"):
        print(f"✅ Proxy configured: {os.getenv('HTTP_PROXY')}")
        return True
    else:
        print("⚠️  No proxy configured (may be OK for some networks)")
        return True

def test_certifi():
    """Test certifi bundle."""
    try:
        import certifi
        bundle_path = certifi.where()
        print(f"✅ Certifi bundle: {bundle_path}")
        return True
    except Exception as e:
        print(f"❌ Certifi import failed: {e}")
        return False

if __name__ == "__main__":
    results = [
        test_certifi(),
        test_ssl(),
        test_proxy()
    ]

    if all(results):
        print("\n✅ All tests passed!")
        sys.exit(0)
    else:
        print("\n❌ Some tests failed")
        sys.exit(1)
```

**Run:**

```bash
python test_setup.py
```

---

## Summary

**Key Takeaways:**

1. **CA Certificates** - Merge corporate CAs into certifi bundle
2. **SSL Environment Variables** - Set SSL_CERT_FILE, REQUESTS_CA_BUNDLE, CURL_CA_BUNDLE
3. **Proxy Configuration** - Set HTTP_PROXY, HTTPS_PROXY, NO_PROXY
4. **Virtual Environments** - Isolate dependencies per project
5. **Automation** - Use setup scripts for reproducibility
6. **Testing** - Verify SSL and proxy work before development

**Checklist:**

- [ ] Corporate CA certificates collected
- [ ] Virtual environment created
- [ ] Certifi bundle updated with corporate CAs
- [ ] SSL environment variables set
- [ ] Proxy environment variables set
- [ ] Dependencies installed successfully
- [ ] Test script passes

## Further Reading

- [Python SSL Documentation](https://docs.python.org/3/library/ssl.html)
- [Requests SSL Cert Verification](https://requests.readthedocs.io/en/latest/user/advanced/#ssl-cert-verification)
- [Certifi Documentation](https://github.com/certifi/python-certifi)
- [Pip Proxy Configuration](https://pip.pypa.io/en/stable/user_guide/#using-a-proxy-server)

---

**Created:** 2026-02-06
**Tags:** #python #enterprise #ssl #certificates #proxy #devops #corporate
