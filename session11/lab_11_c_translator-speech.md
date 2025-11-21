# Lab 11.C: Translator and Speech Services

## Objectives
- Create Translator resource
- Translate text between languages
- Detect language automatically
- Create Speech service
- Convert text to speech
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
RG_NAME="rg-lab11c-translator"
TRANSLATOR_NAME="translator-$RANDOM"
SPEECH_NAME="speech-$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$TRANSLATOR_NAME"
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

## Step 3 – Create Translator Resource

```bash
# Create Translator service
az cognitiveservices account create \
  --name "$TRANSLATOR_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --kind TextTranslation \
  --sku S1 \
  --yes

echo "Translator resource created"
```

---

## Step 4 – Get Translator Credentials

```bash
# Get Translator endpoint
TRANSLATOR_ENDPOINT=$(az cognitiveservices account show \
  --name "$TRANSLATOR_NAME" \
  --resource-group "$RG_NAME" \
  --query properties.endpoint \
  --output tsv)

echo "$TRANSLATOR_ENDPOINT"

# Get Translator key
TRANSLATOR_KEY=$(az cognitiveservices account keys list \
  --name "$TRANSLATOR_NAME" \
  --resource-group "$RG_NAME" \
  --query key1 \
  --output tsv)

echo "Translator credentials retrieved"
```

---

## Step 5 – Install Python Dependencies

```bash
# Install required packages
pip install requests==2.31.0

echo "Python dependencies installed"
```

---

## Step 6 – Create Translation Script

```bash
# Create Python script for translation
cat > translate_text.py << 'EOF'
import os
import requests
import json

key = os.environ['TRANSLATOR_KEY']
endpoint = "https://api.cognitive.microsofttranslator.com"
region = "australiaeast"

def translate_text(text, target_language):
    path = '/translate'
    url = endpoint + path
    
    params = {
        'api-version': '3.0',
        'to': target_language
    }
    
    headers = {
        'Ocp-Apim-Subscription-Key': key,
        'Ocp-Apim-Subscription-Region': region,
        'Content-type': 'application/json'
    }
    
    body = [{'text': text}]
    
    response = requests.post(url, params=params, headers=headers, json=body)
    return response.json()

texts = [
    "Hello, how are you today?",
    "Azure provides powerful cloud services.",
    "Machine learning is transforming industries."
]

target_languages = ['fr', 'es', 'de', 'ja', 'zh-Hans']

print("\nText Translation Results")
print("=" * 60)

for text in texts:
    print(f"\nOriginal: {text}")
    print("-" * 60)
    
    for lang in target_languages:
        result = translate_text(text, lang)
        if result and len(result) > 0:
            translation = result[0]['translations'][0]
            print(f"  {lang}: {translation['text']}")
EOF

echo "Translation script created"
```

---

## Step 7 – Run Translation

```bash
# Set environment variables
export TRANSLATOR_KEY="$TRANSLATOR_KEY"

# Run translation
python3 translate_text.py
```

---

## Step 8 – Create Language Detection Script

```bash
# Create Python script for language detection
cat > detect_language.py << 'EOF'
import os
import requests
import json

key = os.environ['TRANSLATOR_KEY']
endpoint = "https://api.cognitive.microsofttranslator.com"
region = "australiaeast"

def detect_language(text):
    path = '/detect'
    url = endpoint + path
    
    params = {'api-version': '3.0'}
    
    headers = {
        'Ocp-Apim-Subscription-Key': key,
        'Ocp-Apim-Subscription-Region': region,
        'Content-type': 'application/json'
    }
    
    body = [{'text': text}]
    
    response = requests.post(url, params=params, headers=headers, json=body)
    return response.json()

texts = [
    "Hello, how are you?",
    "Bonjour, comment ça va?",
    "Hola, ¿cómo estás?",
    "Guten Tag, wie geht es dir?",
    "こんにちは、元気ですか？"
]

print("\nLanguage Detection Results")
print("=" * 60)

for text in texts:
    result = detect_language(text)
    if result and len(result) > 0:
        lang = result[0]['language']
        score = result[0]['score']
        print(f"\nText: {text}")
        print(f"  Detected language: {lang}")
        print(f"  Confidence: {score:.2f}")
EOF

echo "Language detection script created"
```

---

## Step 9 – Detect Languages

```bash
# Run language detection
python3 detect_language.py
```

---

## Step 10 – Create Speech Service

```bash
# Create Speech service
az cognitiveservices account create \
  --name "$SPEECH_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --kind SpeechServices \
  --sku S0 \
  --yes

echo "Speech service created"
```

---

## Step 11 – Get Speech Credentials

```bash
# Get Speech key
SPEECH_KEY=$(az cognitiveservices account keys list \
  --name "$SPEECH_NAME" \
  --resource-group "$RG_NAME" \
  --query key1 \
  --output tsv)

echo "Speech credentials retrieved"
```

---

## Step 12 – Install Speech SDK

```bash
# Install Azure Speech SDK
pip install azure-cognitiveservices-speech==1.34.0

echo "Speech SDK installed"
```

---

## Step 13 – Create Text-to-Speech Script

```bash
# Create Python script for text-to-speech
cat > text_to_speech.py << 'EOF'
import os
import azure.cognitiveservices.speech as speechsdk

key = os.environ['SPEECH_KEY']
region = "australiaeast"

speech_config = speechsdk.SpeechConfig(subscription=key, region=region)

# Configure audio output to file
audio_config = speechsdk.audio.AudioOutputConfig(filename="output.wav")

# Create synthesizer
synthesizer = speechsdk.SpeechSynthesizer(
    speech_config=speech_config,
    audio_config=audio_config
)

texts = [
    ("Hello, welcome to Azure Speech Services.", "en-US", "en-US-JennyNeural"),
    ("Bonjour, bienvenue aux services vocaux Azure.", "fr-FR", "fr-FR-DeniseNeural"),
    ("Hola, bienvenido a los servicios de voz de Azure.", "es-ES", "es-ES-ElviraNeural")
]

print("\nText-to-Speech Synthesis")
print("=" * 60)

for idx, (text, lang, voice) in enumerate(texts):
    print(f"\nSynthesizing ({lang}): {text}")
    
    # Configure voice
    speech_config.speech_synthesis_voice_name = voice
    
    # Create new synthesizer with updated config
    audio_file = f"output_{idx}.wav"
    audio_config = speechsdk.audio.AudioOutputConfig(filename=audio_file)
    synthesizer = speechsdk.SpeechSynthesizer(
        speech_config=speech_config,
        audio_config=audio_config
    )
    
    # Synthesize
    result = synthesizer.speak_text_async(text).get()
    
    if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
        print(f"  Audio saved to: {audio_file}")
    else:
        print(f"  Error: {result.reason}")
EOF

echo "Text-to-speech script created"
```

---

## Step 14 – Generate Speech

```bash
# Set environment variable
export SPEECH_KEY="$SPEECH_KEY"

# Run text-to-speech
python3 text_to_speech.py

# List generated audio files
ls -lh output_*.wav
```

---

## Step 15 – Create Multi-Language Translation Script

```bash
# Create Python script for multi-language translation
cat > multi_translate.py << 'EOF'
import os
import requests
import json

key = os.environ['TRANSLATOR_KEY']
endpoint = "https://api.cognitive.microsofttranslator.com"
region = "australiaeast"

def translate_to_multiple(text, target_languages):
    path = '/translate'
    url = endpoint + path
    
    params = {
        'api-version': '3.0',
        'to': target_languages
    }
    
    headers = {
        'Ocp-Apim-Subscription-Key': key,
        'Ocp-Apim-Subscription-Region': region,
        'Content-type': 'application/json'
    }
    
    body = [{'text': text}]
    
    response = requests.post(url, params=params, headers=headers, json=body)
    return response.json()

text = "Azure provides comprehensive cloud computing services."
targets = ['fr', 'es', 'de', 'ja', 'zh-Hans', 'ar']

print("\nMulti-Language Translation")
print("=" * 60)
print(f"\nOriginal: {text}\n")

result = translate_to_multiple(text, targets)

if result and len(result) > 0:
    for translation in result[0]['translations']:
        lang = translation['to']
        text = translation['text']
        print(f"  [{lang}] {text}")
EOF

# Run multi-language translation
python3 multi_translate.py
```

---

## Step 16 – Validate Services

```bash
# Check Translator service
az cognitiveservices account show \
  --name "$TRANSLATOR_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, Location:location}" \
  --output table

# Check Speech service
az cognitiveservices account show \
  --name "$SPEECH_NAME" \
  --resource-group "$RG_NAME" \
  --query "{Name:name, SKU:sku.name, Location:location}" \
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
rm -f translate_text.py detect_language.py text_to_speech.py multi_translate.py
rm -f output*.wav

echo "Cleanup complete"
```

---

## Summary

You created Azure Translator service to translate text between multiple languages, detected languages automatically with confidence scores, created Speech service for text-to-speech synthesis, generated audio files in multiple languages with neural voices, and built a multi-language translation pipeline.
