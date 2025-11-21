# Lab 11.B: Text Analytics Sentiment Analysis

## Objectives
- Create Text Analytics resource
- Analyze sentiment of text
- Extract key phrases
- Detect language
- Recognize entities
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
RG_NAME="rg-lab11b-text"
TEXT_NAME="text-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$TEXT_NAME"
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

## Step 3 – Create Text Analytics Resource

```bash
# Create Text Analytics service
az cognitiveservices account create \
  --name "$TEXT_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --kind TextAnalytics \
  --sku S \
  --yes

echo "Text Analytics resource created"
```

---

## Step 4 – Get Endpoint and Key

```bash
# Get Text Analytics endpoint
TEXT_ENDPOINT=$(az cognitiveservices account show \
  --name "$TEXT_NAME" \
  --resource-group "$RG_NAME" \
  --query properties.endpoint \
  --output tsv)

echo "$TEXT_ENDPOINT"

# Get Text Analytics key
TEXT_KEY=$(az cognitiveservices account keys list \
  --name "$TEXT_NAME" \
  --resource-group "$RG_NAME" \
  --query key1 \
  --output tsv)

echo "Text Analytics credentials retrieved"
```

---

## Step 5 – Install Python SDK

```bash
# Install Azure Text Analytics SDK
pip install azure-ai-textanalytics==5.3.0

echo "Python SDK installed"
```

---

## Step 6 – Create Sentiment Analysis Script

```bash
# Create Python script for sentiment analysis
cat > sentiment_analysis.py << 'EOF'
import os
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

endpoint = os.environ['TEXT_ENDPOINT']
key = os.environ['TEXT_KEY']

client = TextAnalyticsClient(endpoint=endpoint, credential=AzureKeyCredential(key))

documents = [
    "I had the best day of my life. The weather was perfect!",
    "This is the worst experience I have ever had. Very disappointed.",
    "The product is okay. Nothing special but it works.",
    "I love this new feature! It makes everything so much easier.",
    "The service was terrible and the staff was rude."
]

print("\nSentiment Analysis Results")
print("=" * 60)

response = client.analyze_sentiment(documents, show_opinion_mining=True)

for idx, result in enumerate(response):
    print(f"\nDocument {idx + 1}: {documents[idx][:50]}...")
    print(f"  Overall sentiment: {result.sentiment}")
    print(f"  Confidence scores:")
    print(f"    Positive: {result.confidence_scores.positive:.2f}")
    print(f"    Neutral: {result.confidence_scores.neutral:.2f}")
    print(f"    Negative: {result.confidence_scores.negative:.2f}")
    
    if result.sentences:
        print(f"  Sentence-level analysis:")
        for sentence in result.sentences:
            print(f"    '{sentence.text[:40]}...' -> {sentence.sentiment}")
EOF

echo "Sentiment analysis script created"
```

---

## Step 7 – Run Sentiment Analysis

```bash
# Set environment variables
export TEXT_ENDPOINT="$TEXT_ENDPOINT"
export TEXT_KEY="$TEXT_KEY"

# Run sentiment analysis
python3 sentiment_analysis.py
```

---

## Step 8 – Create Key Phrase Extraction Script

```bash
# Create Python script for key phrase extraction
cat > key_phrases.py << 'EOF'
import os
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

endpoint = os.environ['TEXT_ENDPOINT']
key = os.environ['TEXT_KEY']

client = TextAnalyticsClient(endpoint=endpoint, credential=AzureKeyCredential(key))

documents = [
    "Microsoft Azure provides cloud computing services including virtual machines, storage, and databases.",
    "Machine learning and artificial intelligence are transforming the technology industry.",
    "The Australian government announced new policies for renewable energy and climate change.",
    "Python is a popular programming language for data science and web development."
]

print("\nKey Phrase Extraction Results")
print("=" * 60)

response = client.extract_key_phrases(documents)

for idx, result in enumerate(response):
    print(f"\nDocument {idx + 1}: {documents[idx][:60]}...")
    if not result.is_error:
        print(f"  Key phrases:")
        for phrase in result.key_phrases:
            print(f"    - {phrase}")
    else:
        print(f"  Error: {result.error.message}")
EOF

echo "Key phrase extraction script created"
```

---

## Step 9 – Extract Key Phrases

```bash
# Run key phrase extraction
python3 key_phrases.py
```

---

## Step 10 – Create Language Detection Script

```bash
# Create Python script for language detection
cat > language_detection.py << 'EOF'
import os
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

endpoint = os.environ['TEXT_ENDPOINT']
key = os.environ['TEXT_KEY']

client = TextAnalyticsClient(endpoint=endpoint, credential=AzureKeyCredential(key))

documents = [
    "Hello, how are you today?",
    "Bonjour, comment allez-vous?",
    "Hola, ¿cómo estás?",
    "Guten Tag, wie geht es Ihnen?",
    "こんにちは、お元気ですか？",
    "你好，你好吗？"
]

print("\nLanguage Detection Results")
print("=" * 60)

response = client.detect_language(documents)

for idx, result in enumerate(response):
    print(f"\nDocument {idx + 1}: {documents[idx]}")
    if not result.is_error:
        print(f"  Detected language: {result.primary_language.name}")
        print(f"  Language code: {result.primary_language.iso6391_name}")
        print(f"  Confidence: {result.primary_language.confidence_score:.2f}")
    else:
        print(f"  Error: {result.error.message}")
EOF

echo "Language detection script created"
```

---

## Step 11 – Detect Languages

```bash
# Run language detection
python3 language_detection.py
```

---

## Step 12 – Create Entity Recognition Script

```bash
# Create Python script for entity recognition
cat > entity_recognition.py << 'EOF'
import os
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

endpoint = os.environ['TEXT_ENDPOINT']
key = os.environ['TEXT_KEY']

client = TextAnalyticsClient(endpoint=endpoint, credential=AzureKeyCredential(key))

documents = [
    "Microsoft was founded by Bill Gates and Paul Allen in Seattle on April 4, 1975.",
    "Amazon Web Services provides cloud computing in multiple regions including Sydney, Australia.",
    "The meeting is scheduled for next Monday at 2:00 PM at the Microsoft office."
]

print("\nEntity Recognition Results")
print("=" * 60)

response = client.recognize_entities(documents)

for idx, result in enumerate(response):
    print(f"\nDocument {idx + 1}: {documents[idx][:60]}...")
    if not result.is_error:
        print(f"  Entities found:")
        for entity in result.entities:
            print(f"    - {entity.text} ({entity.category}")
            if entity.subcategory:
                print(f", {entity.subcategory}")
            print(f") - Confidence: {entity.confidence_score:.2f}")
    else:
        print(f"  Error: {result.error.message}")
EOF

echo "Entity recognition script created"
```

---

## Step 13 – Recognize Entities

```bash
# Run entity recognition
python3 entity_recognition.py
```

---

## Step 14 – Create Batch Analysis Script

```bash
# Create Python script for batch analysis
cat > batch_analysis.py << 'EOF'
import os
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

endpoint = os.environ['TEXT_ENDPOINT']
key = os.environ['TEXT_KEY']

client = TextAnalyticsClient(endpoint=endpoint, credential=AzureKeyCredential(key))

reviews = [
    "The hotel was wonderful. The staff was very friendly and helpful.",
    "Terrible service. The room was dirty and the Wi-Fi didn't work.",
    "Average experience. Nothing special but acceptable for the price.",
    "Absolutely loved it! Will definitely come back again."
]

print("\nBatch Text Analysis")
print("=" * 60)

for idx, review in enumerate(reviews):
    print(f"\nReview {idx + 1}: {review}")
    print("-" * 60)
    
    # Sentiment
    sentiment_result = client.analyze_sentiment([review])[0]
    print(f"  Sentiment: {sentiment_result.sentiment}")
    
    # Key phrases
    phrases_result = client.extract_key_phrases([review])[0]
    print(f"  Key phrases: {', '.join(phrases_result.key_phrases)}")
    
    # Entities
    entities_result = client.recognize_entities([review])[0]
    if entities_result.entities:
        entity_list = [f"{e.text} ({e.category})" for e in entities_result.entities]
        print(f"  Entities: {', '.join(entity_list)}")
EOF

echo "Batch analysis script created"
```

---

## Step 15 – Run Batch Analysis

```bash
# Run batch analysis
python3 batch_analysis.py
```

---

## Step 16 – Validate Service Usage

```bash
# Check Text Analytics account details
az cognitiveservices account show \
  --name "$TEXT_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, Location:location, Endpoint:properties.endpoint}" \
  --output table
```

---

## Step 17 – Cleanup

```bash
# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -f sentiment_analysis.py key_phrases.py language_detection.py entity_recognition.py batch_analysis.py

echo "Cleanup complete"
```

---

## Summary

You created Azure Text Analytics service, analyzed sentiment of text documents with opinion mining, extracted key phrases from content, detected languages with confidence scores, recognized named entities including people, organizations, and locations, and performed batch analysis combining multiple features.
