# Lab 1.C: Host Static Website on Azure Storage

## Overview
This lab demonstrates how to host a static website using Azure Storage Account's built-in static website hosting feature. You'll create a storage account, enable static website hosting, upload HTML/CSS/JavaScript files, and configure custom error pages.

---

## Objectives
- Create Azure Storage Account with static website hosting
- Upload static website files to $web container
- Configure index and error documents
- Test website accessibility via public endpoint
- Configure custom domain (optional)
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
RG_NAME="rg-lab1c-storage"
STORAGE_ACCOUNT="lab1cwebsite$RANDOM"
# Storage account names must be lowercase and unique globally

# Display configuration
echo "LOCATION=$LOCATION"
echo "RG_NAME=$RG_NAME"
echo "STORAGE_ACCOUNT=$STORAGE_ACCOUNT"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group for storage resources
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Storage Account

```bash
# Create storage account with LRS replication
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access true

# Verify storage account creation
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Kind:kind, Sku:sku.name}" \
  --output table
```

---

## Step 4 – Enable Static Website Hosting

```bash
# Enable static website hosting feature
az storage blob service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --static-website \
  --index-document "index.html" \
  --error-document-404-path "404.html"

# Get the primary web endpoint URL
WEBSITE_URL=$(az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "primaryEndpoints.web" \
  --output tsv)

echo "Website URL: $WEBSITE_URL"
```

---

## Step 5 – Create Website Files Locally

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
    <title>Azure Static Website Demo</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <header>
            <h1>☁️ Azure Static Website Hosting</h1>
            <p class="subtitle">Powered by Azure Storage Account</p>
        </header>

        <section class="features">
            <div class="feature-card">
                <h2>🚀 Fast</h2>
                <p>Global CDN distribution for low latency</p>
            </div>
            <div class="feature-card">
                <h2>💰 Cost-Effective</h2>
                <p>Pay only for storage and bandwidth used</p>
            </div>
            <div class="feature-card">
                <h2>🔒 Secure</h2>
                <p>HTTPS by default with custom domains</p>
            </div>
        </section>

        <section class="content">
            <h2>About This Demo</h2>
            <p>This static website is hosted on Azure Blob Storage with static website hosting enabled.</p>
            <p>No servers required - just upload your HTML, CSS, and JavaScript files!</p>
            
            <h3>Features Demonstrated:</h3>
            <ul>
                <li>Static website hosting on Azure Storage</li>
                <li>Custom index and error pages</li>
                <li>Responsive design with CSS</li>
                <li>JavaScript interactivity</li>
            </ul>

            <button onclick="showMessage()">Click Me!</button>
            <p id="message"></p>
        </section>

        <footer>
            <p>Lab 1.C - Azure Cloud Computing</p>
            <p>Deployed: <span id="timestamp"></span></p>
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
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    line-height: 1.6;
    color: #333;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
}

.container {
    max-width: 900px;
    margin: 0 auto;
    background: white;
    border-radius: 12px;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
    overflow: hidden;
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
    font-size: 1.2em;
    opacity: 0.9;
}

.features {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 30px;
    padding: 40px;
    background: #f8f9fa;
}

.feature-card {
    background: white;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    text-align: center;
    transition: transform 0.3s ease;
}

.feature-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.feature-card h2 {
    font-size: 2em;
    margin-bottom: 15px;
}

.content {
    padding: 40px;
}

.content h2 {
    color: #0078d4;
    margin-bottom: 20px;
    font-size: 2em;
}

.content h3 {
    color: #333;
    margin: 30px 0 15px;
}

.content p {
    margin-bottom: 15px;
    color: #555;
}

.content ul {
    margin: 15px 0 30px 40px;
    color: #555;
}

.content li {
    margin-bottom: 10px;
}

button {
    background: #0078d4;
    color: white;
    border: none;
    padding: 12px 30px;
    border-radius: 6px;
    font-size: 16px;
    cursor: pointer;
    transition: background 0.3s ease;
    margin-top: 20px;
}

button:hover {
    background: #005a9e;
}

#message {
    margin-top: 20px;
    font-weight: bold;
    color: #0078d4;
    min-height: 24px;
}

footer {
    background: #2d3748;
    color: white;
    padding: 30px 40px;
    text-align: center;
}

footer p {
    margin: 5px 0;
}

#timestamp {
    color: #00bcf2;
}

@media (max-width: 768px) {
    header h1 {
        font-size: 2em;
    }
    
    .features {
        grid-template-columns: 1fr;
    }
    
    .content {
        padding: 20px;
    }
}
EOF

# Create script.js
cat > script.js << 'EOF'
// Display current timestamp
document.getElementById('timestamp').textContent = new Date().toLocaleString();

// Button click handler
function showMessage() {
    const messages = [
        'Hello from Azure Storage! 🎉',
        'Static websites are awesome! 🚀',
        'No servers needed! ☁️',
        'Scalable and cost-effective! 💰',
        'Deployed in seconds! ⚡'
    ];
    
    const randomMessage = messages[Math.floor(Math.random() * messages.length)];
    document.getElementById('message').textContent = randomMessage;
}

// Console log for demo
console.log('Static website loaded successfully!');
console.log('Hosted on Azure Storage Account');
EOF

# Create 404.html error page
cat > 404.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Page Not Found</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            margin: 0;
        }
        .error-container {
            background: white;
            padding: 60px 40px;
            border-radius: 12px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            text-align: center;
            max-width: 600px;
        }
        h1 {
            font-size: 120px;
            margin: 0;
            color: #0078d4;
        }
        h2 {
            color: #333;
            margin: 20px 0;
        }
        p {
            color: #666;
            margin-bottom: 30px;
        }
        a {
            display: inline-block;
            background: #0078d4;
            color: white;
            padding: 12px 30px;
            text-decoration: none;
            border-radius: 6px;
            transition: background 0.3s;
        }
        a:hover {
            background: #005a9e;
        }
    </style>
</head>
<body>
    <div class="error-container">
        <h1>404</h1>
        <h2>Page Not Found</h2>
        <p>The page you're looking for doesn't exist on this Azure Storage static website.</p>
        <a href="/">Go Home</a>
    </div>
</body>
</html>
EOF

# List created files
echo "Created website files:"
ls -lh
```

---

## Step 6 – Upload Files to Storage

```bash
# Upload all files to $web container (automatically created with static website hosting)
az storage blob upload-batch \
  --account-name "$STORAGE_ACCOUNT" \
  --source "$WEBSITE_DIR" \
  --destination '$web' \
  --auth-mode login \
  --overwrite

# List uploaded files
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --auth-mode login \
  --query "[].{Name:name, Size:properties.contentLength, Type:properties.contentType}" \
  --output table
```

---

## Step 7 – Configure Blob Properties

```bash
# Set content type for CSS file
az storage blob update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "style.css" \
  --content-type "text/css" \
  --auth-mode login

# Set content type for JavaScript file
az storage blob update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "script.js" \
  --content-type "application/javascript" \
  --auth-mode login

# Enable caching headers for better performance
az storage blob update \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "index.html" \
  --content-cache-control "public, max-age=3600" \
  --auth-mode login
```

---

## Step 8 – Test Static Website

```bash
# Display website URL
echo "✅ Website deployed successfully!"
echo "URL: $WEBSITE_URL"

# Test with curl
echo -e "\nTesting website..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" "$WEBSITE_URL"

# Test 404 page
echo -e "\nTesting 404 page..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" "${WEBSITE_URL}nonexistent.html"

# Open in browser
if command -v xdg-open &> /dev/null; then
  xdg-open "$WEBSITE_URL" &
elif [ -n "$BROWSER" ]; then
  "$BROWSER" "$WEBSITE_URL" &
fi
```

---

## Step 9 – Verify Storage Account Configuration

```bash
# Get static website configuration
az storage blob service-properties show \
  --account-name "$STORAGE_ACCOUNT" \
  --auth-mode login \
  --query "staticWebsite" \
  --output table

# Check storage account endpoints
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "primaryEndpoints" \
  --output table

# View storage account properties
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, Location:location, Sku:sku.name, Kind:kind, AccessTier:accessTier}" \
  --output table
```

---

## Step 10 – Update Website Content

```bash
# Update index.html with new content
cd "$WEBSITE_DIR"

sed -i 's/Deployed:/Updated:/g' index.html

# Upload updated file
az storage blob upload \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --name "index.html" \
  --file "index.html" \
  --auth-mode login \
  --overwrite

echo "✅ Website updated - refresh browser to see changes"
```

---

## Step 11 – Monitor Storage Metrics

```bash
# Get storage account usage
az storage account show-usage \
  --location "$LOCATION" \
  --query "{CurrentValue:currentValue, Limit:limit, Unit:unit, Name:name.value}" \
  --output table

# Check blob count and size
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --auth-mode login \
  --query "length(@)"

# Get total size of blobs
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name '$web' \
  --auth-mode login \
  --query "sum([].properties.contentLength)"
```

---

## Step 12 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Verify deletion initiated
az group list --query "[?name=='$RG_NAME']" --output table

# Remove temporary website files
rm -rf "$WEBSITE_DIR"

echo "✅ Cleanup complete - resources will be deleted in background"
```

---

## Summary

**What You Built:**
- Azure Storage Account with static website hosting
- Responsive HTML/CSS/JavaScript website
- Custom 404 error page
- Public web endpoint with HTTPS

**Architecture:**
```
User → HTTPS → Azure Storage ($web container) → Static Files (HTML/CSS/JS)
```

**Key Components:**
- **Storage Account**: Blob storage with static website feature
- **$web Container**: Special container for website files
- **Static Website Hosting**: Serves files directly via HTTP/HTTPS
- **Index Document**: Default page (index.html)
- **Error Document**: Custom 404 page

**What You Learned:**
- Enable static website hosting on Azure Storage
- Upload and manage static website files
- Configure index and error documents
- Set blob content types and caching headers
- Access website via public endpoint
- Update website content dynamically

---

## Best Practices

**Website Hosting:**
- Use Standard tier storage for static websites
- Enable HTTPS by default (automatic)
- Set appropriate cache headers for performance
- Minify CSS and JavaScript files
- Optimize images before upload

**Security:**
- Enable Azure CDN for DDoS protection
- Use custom domain with SSL certificate
- Implement Content Security Policy headers
- Disable public blob access except for $web
- Enable Azure Storage Analytics logging

**Performance:**
- Use Azure CDN for global distribution
- Enable compression for text files
- Set long cache times for static assets
- Use image optimization and lazy loading
- Implement service worker for offline support

**Cost Optimization:**
- Use LRS replication for static content
- Monitor bandwidth usage
- Delete unused files regularly
- Use lifecycle policies for old content
- Consider cool tier for infrequently accessed files

---

## Production Enhancements

**1. Enable Azure CDN**
```bash
# Create CDN profile
az cdn profile create \
  --resource-group "$RG_NAME" \
  --name "cdn-lab1c" \
  --sku Standard_Microsoft \
  --location "$LOCATION"

# Create CDN endpoint
az cdn endpoint create \
  --resource-group "$RG_NAME" \
  --profile-name "cdn-lab1c" \
  --name "lab1c-endpoint" \
  --origin "${STORAGE_ACCOUNT}.z13.web.core.windows.net" \
  --origin-host-header "${STORAGE_ACCOUNT}.z13.web.core.windows.net"
```

**2. Configure Custom Domain**
```bash
# Add custom domain to storage account
az storage account update \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --custom-domain "www.yourdomain.com"

# Verify custom domain
az storage account show \
  --name "$STORAGE_ACCOUNT" \
  --query "customDomain"
```

**3. Enable Logging**
```bash
# Enable diagnostic logging
az monitor diagnostic-settings create \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG_NAME/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT" \
  --name "storage-logs" \
  --logs '[{"category":"StorageRead","enabled":true},{"category":"StorageWrite","enabled":true}]'
```

**4. Implement SPA Routing**
```bash
# For single-page applications, set error document to index.html
az storage blob service-properties update \
  --account-name "$STORAGE_ACCOUNT" \
  --static-website \
  --index-document "index.html" \
  --error-document-404-path "index.html"
```

---

## Troubleshooting

**Website not accessible:**
- Verify static website hosting is enabled
- Check blob public access is allowed on storage account
- Confirm files are uploaded to $web container
- Test URL format: `https://<account>.z13.web.core.windows.net/`
- Check browser console for errors

**404 errors for CSS/JS files:**
- Verify files are uploaded to $web container
- Check file names match references in HTML
- Confirm content-type is set correctly
- Test file URLs directly in browser
- Check for case sensitivity in file names

**Slow website loading:**
- Enable Azure CDN for caching
- Optimize images (compress, resize)
- Minify CSS and JavaScript files
- Set appropriate cache headers
- Use browser developer tools to identify bottlenecks

**Cannot upload files:**
- Check Azure CLI authentication: `az login`
- Verify contributor role on storage account
- Try using `--auth-mode key` with connection string
- Check firewall rules on storage account
- Verify storage account exists and is accessible

**Custom domain not working:**
- Verify CNAME record points to storage endpoint
- Allow time for DNS propagation (up to 48 hours)
- Check domain ownership verification
- Use CDN for custom domain with HTTPS
- Test with nslookup or dig command

---

## Additional Resources

- [Azure Storage Static Website Hosting](https://docs.microsoft.com/azure/storage/blobs/storage-blob-static-website)
- [Azure CDN Documentation](https://docs.microsoft.com/azure/cdn/)
- [Storage Account Overview](https://docs.microsoft.com/azure/storage/common/storage-account-overview)
- [Custom Domains for Storage](https://docs.microsoft.com/azure/storage/blobs/storage-custom-domain-name)
- [Azure Storage CLI Reference](https://docs.microsoft.com/cli/azure/storage)
