# Lab 11.A: Computer Vision Analysis

## Objectives
- Create Computer Vision resource
- Analyze images for objects and tags
- Extract text from images (OCR)
- Detect faces and landmarks
- Generate image descriptions
- Clean up all resources

---

## Prerequisites
- Azure CLI installed (`az --version`)
- Azure subscription with contributor access
- Python 3.8+ installed
- Location: Australia East

---

## Step 1 – Set Variables

```bash
# Set Azure region and resource naming
LOCATION="australiaeast"
RG_NAME="rg-lab11a-vision"
VISION_NAME="vision-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$VISION_NAME"
```

---

## Step 2 – Create Resource Group

```bash
# Create resource group
az group create \
  --name "$RG_NAME" \
  --location "$LOCATION"
```

---

## Step 3 – Create Computer Vision Resource

```bash
# Create Computer Vision service
az cognitiveservices account create \
  --name "$VISION_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --kind ComputerVision \
  --sku S1 \
  --yes

echo "Computer Vision resource created"
```

---

## Step 4 – Get Endpoint and Key

```bash
# Get Computer Vision endpoint
VISION_ENDPOINT=$(az cognitiveservices account show \
  --name "$VISION_NAME" \
  --resource-group "$RG_NAME" \
  --query properties.endpoint \
  --output tsv)

echo "$VISION_ENDPOINT"

# Get Computer Vision key
VISION_KEY=$(az cognitiveservices account keys list \
  --name "$VISION_NAME" \
  --resource-group "$RG_NAME" \
  --query key1 \
  --output tsv)

echo "Computer Vision credentials retrieved"
```

---

## Step 5 – Install Python SDK

```bash
# Install Azure Computer Vision SDK
pip install azure-cognitiveservices-vision-computervision==0.9.0
pip install Pillow

echo "Python SDK installed"
```

---

## Step 6 – Download Sample Images

```bash
# Create images directory
mkdir -p images

# Download sample images
curl -o images/city.jpg "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/ComputerVision/Images/landmark.jpg"
curl -o images/text.jpg "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/ComputerVision/Images/printed_text.jpg"
curl -o images/faces.jpg "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/Face/Images/Family1-Dad1.jpg"

echo "Sample images downloaded"
```

---

## Step 7 – Create Image Analysis Script

```bash
# Create Python script for image analysis
cat > analyze_image.py << 'EOF'
import os
import sys
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

endpoint = os.environ['VISION_ENDPOINT']
key = os.environ['VISION_KEY']

client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(key))

image_path = sys.argv[1] if len(sys.argv) > 1 else 'images/city.jpg'

print(f"\nAnalyzing image: {image_path}")
print("=" * 60)

with open(image_path, 'rb') as image_stream:
    # Analyze image
    features = ['categories', 'description', 'tags', 'objects', 'brands', 'faces', 'color']
    analysis = client.analyze_image_in_stream(image_stream, visual_features=features)
    
    # Description
    print("\nDescription:")
    for caption in analysis.description.captions:
        print(f"  {caption.text} (confidence: {caption.confidence:.2f})")
    
    # Tags
    print("\nTags:")
    for tag in analysis.tags:
        print(f"  {tag.name} (confidence: {tag.confidence:.2f})")
    
    # Objects
    if analysis.objects:
        print("\nObjects detected:")
        for obj in analysis.objects:
            print(f"  {obj.object_property} at ({obj.rectangle.x}, {obj.rectangle.y})")
    
    # Faces
    if analysis.faces:
        print("\nFaces detected:")
        for face in analysis.faces:
            print(f"  Age: {face.age}, Gender: {face.gender}")
    
    # Colors
    print("\nColor analysis:")
    print(f"  Dominant colors: {', '.join(analysis.color.dominant_colors)}")
    print(f"  Accent color: #{analysis.color.accent_color}")
    print(f"  Is B&W: {analysis.color.is_bw_img}")
EOF

echo "Image analysis script created"
```

---

## Step 8 – Analyze Sample Images

```bash
# Set environment variables
export VISION_ENDPOINT="$VISION_ENDPOINT"
export VISION_KEY="$VISION_KEY"

# Analyze city image
python3 analyze_image.py images/city.jpg

# Analyze faces image
python3 analyze_image.py images/faces.jpg
```

---

## Step 9 – Create OCR Script

```bash
# Create Python script for text extraction
cat > extract_text.py << 'EOF'
import os
import sys
import time
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

endpoint = os.environ['VISION_ENDPOINT']
key = os.environ['VISION_KEY']

client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(key))

image_path = sys.argv[1] if len(sys.argv) > 1 else 'images/text.jpg'

print(f"\nExtracting text from: {image_path}")
print("=" * 60)

with open(image_path, 'rb') as image_stream:
    # Start read operation
    read_operation = client.read_in_stream(image_stream, raw=True)
    operation_location = read_operation.headers['Operation-Location']
    operation_id = operation_location.split('/')[-1]
    
    # Wait for operation to complete
    while True:
        result = client.get_read_result(operation_id)
        if result.status.lower() not in ['notstarted', 'running']:
            break
        time.sleep(1)
    
    # Display results
    if result.status.lower() == 'succeeded':
        print("\nText extracted:")
        for page in result.analyze_result.read_results:
            for line in page.lines:
                print(f"  {line.text}")
                print(f"    Bounding box: {line.bounding_box}")
EOF

echo "OCR script created"
```

---

## Step 10 – Extract Text from Images

```bash
# Extract text from image
python3 extract_text.py images/text.jpg
```

---

## Step 11 – Create Thumbnail Generation Script

```bash
# Create Python script for thumbnail generation
cat > generate_thumbnail.py << 'EOF'
import os
import sys
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

endpoint = os.environ['VISION_ENDPOINT']
key = os.environ['VISION_KEY']

client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(key))

image_path = sys.argv[1] if len(sys.argv) > 1 else 'images/city.jpg'
output_path = 'images/thumbnail.jpg'

print(f"\nGenerating thumbnail for: {image_path}")

with open(image_path, 'rb') as image_stream:
    # Generate smart-cropped thumbnail
    thumbnail = client.generate_thumbnail_in_stream(
        width=200,
        height=200,
        image=image_stream,
        smart_cropping=True
    )
    
    # Save thumbnail
    with open(output_path, 'wb') as thumb_file:
        for chunk in thumbnail:
            thumb_file.write(chunk)
    
    print(f"Thumbnail saved to: {output_path}")
EOF

echo "Thumbnail generation script created"
```

---

## Step 12 – Generate Thumbnails

```bash
# Generate thumbnail
python3 generate_thumbnail.py images/city.jpg

# Verify thumbnail created
ls -lh images/thumbnail.jpg
```

---

## Step 13 – Analyze Image from URL

```bash
# Create Python script for URL analysis
cat > analyze_url.py << 'EOF'
import os
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

endpoint = os.environ['VISION_ENDPOINT']
key = os.environ['VISION_KEY']

client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(key))

image_url = "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/ComputerVision/Images/landmark.jpg"

print(f"\nAnalyzing image from URL")
print("=" * 60)

# Analyze image
features = ['description', 'tags', 'categories']
analysis = client.analyze_image(image_url, visual_features=features)

print("\nDescription:")
for caption in analysis.description.captions:
    print(f"  {caption.text} (confidence: {caption.confidence:.2f})")

print("\nTop tags:")
for tag in analysis.tags[:5]:
    print(f"  {tag.name} (confidence: {tag.confidence:.2f})")
EOF

# Run URL analysis
python3 analyze_url.py
```

---

## Step 14 – Test Different Image Features

```bash
# Create comprehensive test script
cat > test_features.py << 'EOF'
import os
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

endpoint = os.environ['VISION_ENDPOINT']
key = os.environ['VISION_KEY']

client = ComputerVisionClient(endpoint, CognitiveServicesCredentials(key))

image_url = "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/ComputerVision/Images/celebrities.jpg"

print("\nTesting domain-specific models")
print("=" * 60)

# Analyze celebrities
result = client.analyze_image_by_domain('celebrities', image_url)
if result.result.get('celebrities'):
    print("\nCelebrities detected:")
    for celebrity in result.result['celebrities']:
        print(f"  {celebrity['name']} (confidence: {celebrity['confidence']:.2f})")

# Analyze landmarks
landmark_url = "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-sample-data-files/master/ComputerVision/Images/landmark.jpg"
result = client.analyze_image_by_domain('landmarks', landmark_url)
if result.result.get('landmarks'):
    print("\nLandmarks detected:")
    for landmark in result.result['landmarks']:
        print(f"  {landmark['name']} (confidence: {landmark['confidence']:.2f})")
EOF

# Run feature tests
python3 test_features.py
```

---

## Step 15 – Validate Service Usage

```bash
# Check Computer Vision account details
az cognitiveservices account show \
  --name "$VISION_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, Location:location, Endpoint:properties.endpoint}" \
  --output table
```

---

## Step 16 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf images
rm -f analyze_image.py extract_text.py generate_thumbnail.py analyze_url.py test_features.py

echo "Cleanup complete"
```

---

## Summary

You created Azure Computer Vision service, analyzed images to detect objects, tags, faces, and colors, extracted text using OCR capabilities, generated smart-cropped thumbnails, and explored domain-specific models for celebrities and landmarks.
