# Lab 3.C: Static Website + Azure CDN

## Overview
This lab demonstrates hosting a static website on Azure Storage with Azure CDN for global content delivery. You'll create a static website, configure Azure CDN for caching and acceleration, set up custom domains, and implement caching rules.

---

## Objectives
- Host static website on Azure Storage
- Create Azure CDN profile and endpoint
- Configure CDN caching rules and TTL
- Test content delivery and caching
- Purge CDN cached content
- Monitor CDN metrics and performance
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Basic HTML/CSS knowledge
- Web browser for testing
- Location: East US

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="eastus"
RG_NAME="rg-lab3c-cdn"
STORAGE_ACCOUNT="lab3ccdn$RANDOM"
CDN_PROFILE="cdn-lab3c"
CDN_ENDPOINT="lab3c-endpoint-$RANDOM"

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "STORAGE_ACCOUNT=$STORAGE_ACCOUNT"
echo "CDN_PROFILE=$CDN_PROFILE"
echo "CDN_ENDPOINT=$CDN_ENDPOINT"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for CDN resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Storage Account for Static Website

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access true

# Enable static website hosting
az storage blob service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --static-website \
  --index-document "index.html" \
  --error-document-404-path "404.html"

# Get static website URL
WEBSITE_URL=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "primaryEndpoints.web" \
  --output tsv)

echo "Static Website URL: $WEBSITE_URL"
```

---

## Step 4 – Create Website Content

```bash
# Create temporary directory for website files
WEBSITE_DIR=$(mktemp -d)
cd "$WEBSITE_DIR"

# Create index.html
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Azure CDN Demo</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>🚀 Azure CDN + Static Website</h1>
            <p class="subtitle">Globally Distributed Content Delivery</p>
        </header>

        <section class="info">
            <div class="card">
                <h2>🌍 Global Reach</h2>
                <p>Content delivered from 100+ edge locations worldwide</p>
            </div>
            <div class="card">
                <h2>⚡ Fast Performance</h2>
                <p>Cached content served from nearest POP</p>
            </div>
            <div class="card">
                <h2>📊 Scalable</h2>
                <p>Handle traffic spikes effortlessly</p>
            </div>
        </section>

        <section class="demo">
            <h2>CDN Features Demonstrated</h2>
            <ul>
                <li>Static content caching</li>
                <li>Custom caching rules</li>
                <li>Cache purging</li>
                <li>HTTPS delivery</li>
                <li>Compression</li>
            </ul>
            
            <div class="test">
                <h3>Test CDN Caching</h3>
                <p>Load Time: <span id="loadTime">Calculating...</span></p>
                <p>Timestamp: <span id="timestamp"></span></p>
                <button onclick="location.reload()">Refresh Page</button>
            </div>
        </section>

        <footer>
            <p>Lab 3.C - Azure Cloud Computing</p>
            <p>Served via Azure CDN</p>
        </footer>
    </div>

    <script src="script.js"></script>
</body>
</html>
EOF

# Create style.css
cat > style.css << 'EOF'
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    background: white;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

header {
    background: linear-gradient(135deg, #0078d4 0%, #00bcf2 100%);
    color: white;
    padding: 60px 40px;
    text-align: center;
}

header h1 {
    font-size: 3em;
    margin-bottom: 10px;
}

.subtitle {
    font-size: 1.3em;
    opacity: 0.9;
}

.info {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 30px;
    padding: 40px;
    background: #f8f9fa;
}

.card {
    background: white;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    text-align: center;
    transition: transform 0.3s;
}

.card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 12px rgba(0, 0, 0, 0.15);
}

.card h2 {
    color: #0078d4;
    margin-bottom: 15px;
    font-size: 1.8em;
}

.demo {
    padding: 40px;
}

.demo h2 {
    color: #0078d4;
    margin-bottom: 20px;
    font-size: 2em;
}

.demo ul {
    list-style-position: inside;
    margin: 20px 0;
    color: #555;
}

.demo li {
    margin: 10px 0;
    padding: 10px;
    background: #f0f0f0;
    border-radius: 4px;
}

.test {
    background: #e3f2fd;
    padding: 30px;
    border-radius: 8px;
    margin-top: 30px;
}

.test h3 {
    color: #0078d4;
    margin-bottom: 20px;
}

.test p {
    margin: 10px 0;
    font-size: 1.1em;
}

button {
    background: #0078d4;
    color: white;
    border: none;
    padding: 12px 30px;
    border-radius: 6px;
    font-size: 16px;
    cursor: pointer;
    margin-top: 20px;
    transition: background 0.3s;
}

button:hover {
    background: #005a9e;
}

footer {
    background: #2d3748;
    color: white;
    padding: 30px;
    text-align: center;
}

footer p {
    margin: 5px 0;
}

@media (max-width: 768px) {
    header h1 {
        font-size: 2em;
    }
    
    .info {
        grid-template-columns: 1fr;
    }
}
EOF

# Create script.js
cat > script.js << 'EOF'
const loadStart = performance.timing.navigationStart;
const loadEnd = performance.timing.loadEventEnd;

window.addEventListener('load', function() {
    const loadTime = loadEnd - loadStart;
    document.getElementById('loadTime').textContent = loadTime + ' ms';
});

document.getElementById('timestamp').textContent = new Date().toLocaleString();

console.log('Website loaded via Azure CDN');
console.log('Cache headers should be present in Network tab');
EOF

# Create 404 error page
cat > 404.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Not Found</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            margin: 0;
        }
        .error {
            background: white;
            padding: 60px;
            border-radius: 12px;
            text-align: center;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
        }
        h1 {
            font-size: 120px;
            color: #0078d4;
            margin: 0;
        }
        h2 {
            color: #333;
            margin: 20px 0;
        }
        a {
            color: #0078d4;
            text-decoration: none;
            font-size: 1.2em;
        }
    </style>
</head>
<body>
    <div class="error">
        <h1>404</h1>
        <h2>Page Not Found</h2>
        <p>The requested content doesn't exist.</p>
        <br>
        <a href="/">← Go Home</a>
    </div>
</body>
</html>
EOF

# Create large image for testing caching
convert -size 1920x1080 gradient:blue-purple large-image.jpg 2>/dev/null || \
echo "Sample large image content" > large-image.jpg

ls -lh
```

---

## Step 5 – Upload Website to Storage

```bash
# Upload all files to $web container
az storage blob upload-batch \
  --account-name "$STORAGE_ACCOUNT" \
  --source "$WEBSITE_DIR" \
  --destination '$web' \
  --auth-mode login \
  --overwrite

# Set content types
az storage blob update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "style.css" \
  --content-type "text/css" \
  --auth-mode login

az storage blob update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "script.js" \
  --content-type "application/javascript" \
  --auth-mode login

echo "✅ Website uploaded to storage"
echo "Website URL: $WEBSITE_URL"
```

---

## Step 6 – Create Azure CDN Profile

```bash
# Create CDN profile (Standard Microsoft tier)
az cdn profile create \
  --name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_Microsoft

# Verify CDN profile creation
az cdn profile show \
  --name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Sku:sku.name, State:provisioningState}" \
  --output table
```

---

## Step 7 – Create CDN Endpoint

```bash
# Extract origin hostname from storage URL
ORIGIN_HOST=$(echo $WEBSITE_URL | sed 's|https://||' | sed 's|/||')
echo "Origin Host: $ORIGIN_HOST"

# Create CDN endpoint
az cdn endpoint create \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --origin "$ORIGIN_HOST" \
  --origin-host-header "$ORIGIN_HOST" \
  --enable-compression true \
  --content-types-to-compress "text/html" "text/css" "application/javascript" "application/json"

# Get CDN endpoint URL
CDN_URL=$(az cdn endpoint show \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "hostName" \
  --output tsv)

echo "CDN Endpoint URL: https://$CDN_URL"
```

---

## Step 8 – Configure CDN Caching Rules

```bash
# Set caching rules for different content types
# Note: Standard Microsoft CDN uses standard rules engine

# Update endpoint with query string caching
az cdn endpoint update \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query-string-caching-behavior IgnoreQueryString

# Verify caching configuration
az cdn endpoint show \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, CacheQuery:queryStringCachingBehavior, Compression:isCompressionEnabled}" \
  --output table
```

---

## Step 9 – Test CDN Endpoint

```bash
# Wait for CDN endpoint to be ready
echo "⏳ Waiting for CDN endpoint to propagate (2-3 minutes)..."
sleep 180

# Test CDN endpoint
echo -e "\n=== Testing CDN Endpoint ==="
curl -I "https://$CDN_URL"

# Check for CDN cache headers
echo -e "\n=== Checking Cache Headers ==="
curl -s -I "https://$CDN_URL" | grep -i "cache\|cdn\|x-"

# Test specific file
echo -e "\n=== Testing CSS File ==="
curl -I "https://$CDN_URL/style.css"

# Open in browser
if command -v xdg-open &> /dev/null; then
  xdg-open "https://$CDN_URL" &
elif [ -n "$BROWSER" ]; then
  "$BROWSER" "https://$CDN_URL" &
fi
```

---

## Step 10 – Purge CDN Cache

```bash
# Purge all cached content
az cdn endpoint purge \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --content-paths '/*'

echo "✅ CDN cache purged"

# Purge specific files
az cdn endpoint purge \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --content-paths '/index.html' '/style.css' '/script.js'

echo "✅ Specific files purged from cache"
```

---

## Step 11 – Update Website Content

```bash
# Update index.html
cd "$WEBSITE_DIR"
sed -i 's/Globally Distributed/Globally Distributed - UPDATED/g' index.html

# Upload updated file
az storage blob upload \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "index.html" \
  --file "index.html" \
  --auth-mode login \
  --overwrite

# Purge CDN cache to serve new content
az cdn endpoint purge \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --content-paths '/index.html'

echo "✅ Website updated - refresh browser after 1-2 minutes"
```

---

## Step 12 – Configure HTTPS and Custom Domain (Optional)

```bash
# Enable HTTPS (already enabled by default on azureedge.net)
az cdn endpoint show \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "isHttpsAllowed" \
  --output tsv

# To add custom domain (requires DNS configuration):
# 1. Create CNAME record: www.yourdomain.com -> $CDN_ENDPOINT.azureedge.net
# 2. Add custom domain to CDN endpoint:
# az cdn custom-domain create \
#   --endpoint-name "$CDN_ENDPOINT" \
#   --profile-name "$CDN_PROFILE" \
#   --resource-group "$RG_NAME" \
#   --hostname "www.yourdomain.com" \
#   --name "custom-domain"

echo "HTTPS is enabled by default on Azure CDN"
```

---

## Step 13 – Monitor CDN Metrics

```bash
# Get CDN endpoint metrics
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Cdn/profiles/$CDN_PROFILE/endpoints/$CDN_ENDPOINT" \
  --metric "BytesServed" "RequestCount" \
  --start-time "$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  --interval PT1M \
  --output table

# List available metrics
az monitor metrics list-definitions \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Cdn/profiles/$CDN_PROFILE/endpoints/$CDN_ENDPOINT" \
  --query "[].{Metric:name.value, Unit:unit}" \
  --output table
```

---

## Step 14 – View CDN Endpoint Details

```bash
# Get full endpoint configuration
az cdn endpoint show \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --output json | python3 -m json.tool

# List all CDN endpoints
az cdn endpoint list \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, HostName:hostName, State:resourceState}" \
  --output table
```

---

## Step 15 – Cleanup

```bash
# Delete CDN endpoint
az cdn endpoint delete \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --yes

# Delete CDN profile
az cdn profile delete \
  --name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --yes

# Delete resource group
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local website files
rm -rf "$WEBSITE_DIR"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Static website hosted on Azure Storage
- Azure CDN profile and endpoint
- Global content delivery network
- Caching rules and compression
- HTTPS-enabled CDN endpoint

**Architecture:**
```
User Request → CDN Edge POP (Cache Hit) → Content Delivered
                    ↓ (Cache Miss)
                Origin (Storage) → CDN Edge → Cached & Delivered
```

**Key Components:**
- **Azure Storage Static Website**: Origin server for content
- **Azure CDN Profile**: CDN service configuration
- **CDN Endpoint**: Entry point for content delivery
- **Edge Locations**: Geographically distributed cache servers
- **Caching Rules**: Control how content is cached

**What You Learned:**
- Host static websites on Azure Storage
- Configure Azure CDN for content delivery
- Set caching rules and compression
- Purge cached content
- Monitor CDN performance
- Update content with cache invalidation

---

## Best Practices

**CDN Configuration:**
- Enable compression for text-based files
- Configure appropriate cache TTL values
- Use query string caching wisely
- Implement custom domains with HTTPS
- Monitor CDN metrics regularly

**Caching Strategy:**
- Cache static assets aggressively (long TTL)
- Use versioned URLs for cache busting
- Purge cache when content changes
- Test caching behavior before production
- Implement cache-control headers

**Performance:**
- Optimize images before upload
- Minify CSS and JavaScript
- Use CDN for all static assets
- Enable HTTP/2 support
- Leverage browser caching

**Cost Optimization:**
- Use appropriate CDN tier (Standard vs Premium)
- Monitor bandwidth usage
- Implement lifecycle policies for old content
- Use compression to reduce transfer size
- Review and optimize cache hit ratio

---

## Production Enhancements

**1. Implement Custom Caching Rules**
```bash
# For Premium Verizon CDN (supports rules engine)
cat > caching-rules.json << 'EOF'
{
  "rules": [
    {
      "name": "cache-images",
      "order": 1,
      "conditions": [{"matchType": "UrlFileExtension", "matchValues": ["jpg", "png", "gif"]}],
      "actions": [{"actionType": "CacheExpiration", "cacheDuration": "7.00:00:00"}]
    }
  ]
}
EOF
```

**2. Configure WAF (Web Application Firewall)**
```bash
# Create WAF policy (Premium tier required)
az network front-door waf-policy create \
  --name "waf-policy" \
  --resource-group "$RG_NAME" \
  --sku Premium_AzureFrontDoor
```

**3. Enable Diagnostic Logging**
```bash
# Configure diagnostic settings
az monitor diagnostic-settings create \
  --name "cdn-diagnostics" \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Cdn/profiles/$CDN_PROFILE" \
  --logs '[{"category":"CoreAnalytics","enabled":true}]' \
  --workspace "/subscriptions/.../workspaces/your-workspace"
```

**4. Implement Geo-Filtering**
```bash
# Restrict access by country (Premium tier)
az cdn endpoint update \
  --name "$CDN_ENDPOINT" \
  --profile-name "$CDN_PROFILE" \
  --resource-group "$RG_NAME" \
  --geo-filter-actions Allow \
  --geo-filter-country-codes US CA GB
```

---

## Troubleshooting

**CDN endpoint not accessible:**
- Wait for propagation (up to 10 minutes)
- Verify origin is accessible
- Check DNS resolution of endpoint
- Review CDN endpoint status
- Confirm HTTPS is enabled

**Content not updating:**
- Purge CDN cache
- Check cache-control headers on origin
- Verify query string caching settings
- Use versioned URLs for immediate updates
- Wait for TTL expiration

**Slow performance:**
- Check CDN edge location coverage
- Monitor cache hit ratio
- Review origin server performance
- Enable compression
- Optimize file sizes

**High CDN costs:**
- Review bandwidth usage metrics
- Check for cache misses (low hit ratio)
- Optimize caching rules
- Implement longer TTL where appropriate
- Consider CDN tier pricing

**HTTPS certificate issues:**
- Verify custom domain DNS configuration
- Allow time for certificate provisioning
- Check certificate renewal status
- Review domain validation requirements
- Use Azure-managed certificates

---

## Additional Resources

- [Azure CDN Documentation](https://docs.microsoft.com/azure/cdn/)
- [Static Website Hosting](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website)
- [CDN Caching Behavior](https://docs.microsoft.com/azure/cdn/cdn-how-caching-works)
- [Custom Domains on CDN](https://docs.microsoft.com/azure/cdn/cdn-map-content-to-custom-domain)
- [Azure CDN Pricing](https://azure.microsoft.com/pricing/details/cdn/)
