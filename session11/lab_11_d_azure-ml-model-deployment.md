# Lab 11.D: Azure ML Model Deployment

## Objectives
- Create Azure Machine Learning workspace
- Create and train a simple model
- Register model in workspace
- Deploy model as web service
- Test model endpoint
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
RG_NAME="rg-lab11d-ml"
WORKSPACE_NAME="ml-workspace-$RANDOM"
STORAGE_ACCOUNT="mlstorage$RANDOM"

# Display configuration
echo "$LOCATION"
echo "$RG_NAME"
echo "$WORKSPACE_NAME"
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

## Step 3 – Install Azure ML Extension

```bash
# Install Azure ML CLI extension
az extension add --name ml --yes

echo "Azure ML extension installed"
```

---

## Step 4 – Create Storage Account

```bash
# Create storage account
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --sku Standard_LRS

echo "Storage account created"
```

---

## Step 5 – Create ML Workspace

```bash
# Create Azure ML workspace
az ml workspace create \
  --name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --location "$LOCATION" \
  --storage-account "$STORAGE_ACCOUNT"

echo "ML workspace created"
```

---

## Step 6 – Install Python ML SDK

```bash
# Install Azure ML SDK
pip install azureml-core==1.54.0
pip install scikit-learn==1.3.0
pip install joblib==1.3.2

echo "Python ML SDK installed"
```

---

## Step 7 – Create Training Script

```bash
# Create directory for ML code
mkdir -p ml-code

# Create training script
cat > ml-code/train.py << 'EOF'
import os
import joblib
from sklearn.datasets import load_iris
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

print("Loading iris dataset...")
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42
)

print("Training decision tree classifier...")
model = DecisionTreeClassifier(max_depth=3, random_state=42)
model.fit(X_train, y_train)

print("Evaluating model...")
y_pred = model.predict(X_test)
accuracy = accuracy_score(y_test, y_pred)
print(f"Model accuracy: {accuracy:.2f}")

print("Saving model...")
os.makedirs('outputs', exist_ok=True)
model_path = 'outputs/iris_model.pkl'
joblib.dump(model, model_path)
print(f"Model saved to: {model_path}")
EOF

echo "Training script created"
```

---

## Step 8 – Train Model Locally

```bash
# Train model locally
cd ml-code
python3 train.py
cd ..

echo "Model trained"
```

---

## Step 9 – Create Workspace Connection Script

```bash
# Create Python script to connect to workspace
cat > connect_workspace.py << EOF
from azureml.core import Workspace

ws = Workspace(
    subscription_id="$(az account show --query id -o tsv)",
    resource_group="$RG_NAME",
    workspace_name="$WORKSPACE_NAME"
)

print(f"Workspace name: {ws.name}")
print(f"Workspace region: {ws.location}")
print(f"Resource group: {ws.resource_group}")

# Save workspace config
ws.write_config(path=".")
print("Workspace config saved")
EOF

echo "Workspace connection script created"
```

---

## Step 10 – Connect to Workspace

```bash
# Connect to workspace and save config
python3 connect_workspace.py

echo "Connected to workspace"
```

---

## Step 11 – Register Model

```bash
# Create model registration script
cat > register_model.py << 'EOF'
from azureml.core import Workspace, Model

ws = Workspace.from_config()

print("Registering model...")
model = Model.register(
    workspace=ws,
    model_name="iris-classifier",
    model_path="ml-code/outputs/iris_model.pkl",
    description="Decision tree classifier for iris dataset",
    tags={"type": "classification", "framework": "scikit-learn"}
)

print(f"Model registered: {model.name}")
print(f"Model version: {model.version}")
print(f"Model ID: {model.id}")
EOF

# Register model
python3 register_model.py
```

---

## Step 12 – Create Scoring Script

```bash
# Create scoring script for deployment
cat > ml-code/score.py << 'EOF'
import json
import joblib
import numpy as np
from azureml.core.model import Model

def init():
    global model
    model_path = Model.get_model_path('iris-classifier')
    model = joblib.load(model_path)
    print("Model loaded successfully")

def run(raw_data):
    try:
        data = json.loads(raw_data)['data']
        data = np.array(data)
        predictions = model.predict(data)
        class_names = ['setosa', 'versicolor', 'virginica']
        result = [class_names[pred] for pred in predictions]
        return json.dumps({"predictions": result})
    except Exception as e:
        return json.dumps({"error": str(e)})
EOF

echo "Scoring script created"
```

---

## Step 13 – Create Environment File

```bash
# Create conda environment file
cat > ml-code/environment.yml << 'EOF'
name: iris-env
dependencies:
  - python=3.8
  - pip:
    - azureml-defaults
    - scikit-learn==1.3.0
    - joblib==1.3.2
EOF

echo "Environment file created"
```

---

## Step 14 – Create Deployment Script

```bash
# Create deployment script
cat > deploy_model.py << 'EOF'
from azureml.core import Workspace, Model, Environment
from azureml.core.webservice import AciWebservice
from azureml.core.model import InferenceConfig

ws = Workspace.from_config()

print("Getting registered model...")
model = Model(ws, name='iris-classifier')

print("Creating environment...")
env = Environment.from_conda_specification(
    name='iris-env',
    file_path='ml-code/environment.yml'
)

print("Creating inference config...")
inference_config = InferenceConfig(
    entry_script='ml-code/score.py',
    environment=env
)

print("Creating deployment configuration...")
deployment_config = AciWebservice.deploy_configuration(
    cpu_cores=1,
    memory_gb=1,
    tags={'model': 'iris-classifier', 'type': 'demo'},
    description='Iris classification model'
)

print("Deploying model...")
service = Model.deploy(
    workspace=ws,
    name='iris-service',
    models=[model],
    inference_config=inference_config,
    deployment_config=deployment_config,
    overwrite=True
)

service.wait_for_deployment(show_output=True)

print(f"\nService name: {service.name}")
print(f"Service state: {service.state}")
print(f"Scoring URI: {service.scoring_uri}")
EOF

echo "Deployment script created"
```

---

## Step 15 – Deploy Model

```bash
# Deploy model (this may take 5-10 minutes)
echo "Deploying model... (this may take several minutes)"
python3 deploy_model.py

echo "Model deployed"
```

---

## Step 16 – Test Deployed Model

```bash
# Create test script
cat > test_service.py << 'EOF'
import json
from azureml.core import Workspace
from azureml.core.webservice import Webservice

ws = Workspace.from_config()

print("Getting service...")
service = Webservice(workspace=ws, name='iris-service')

print(f"Service state: {service.state}")
print(f"Scoring URI: {service.scoring_uri}")

# Test data (sepal length, sepal width, petal length, petal width)
test_samples = [
    [5.1, 3.5, 1.4, 0.2],  # Should be setosa
    [6.7, 3.1, 4.7, 1.5],  # Should be versicolor
    [6.3, 2.9, 5.6, 1.8]   # Should be virginica
]

print("\nTesting model predictions:")
print("=" * 60)

for idx, sample in enumerate(test_samples):
    input_data = json.dumps({'data': [sample]})
    result = service.run(input_data)
    prediction = json.loads(result)
    print(f"\nSample {idx + 1}: {sample}")
    print(f"Prediction: {prediction}")
EOF

# Test service
python3 test_service.py
```

---

## Step 17 – Get Service Logs

```bash
# Create script to get service logs
cat > get_logs.py << 'EOF'
from azureml.core import Workspace
from azureml.core.webservice import Webservice

ws = Workspace.from_config()
service = Webservice(workspace=ws, name='iris-service')

print("Service logs:")
print("=" * 60)
print(service.get_logs())
EOF

# Get logs
python3 get_logs.py
```

---

## Step 18 – List Workspace Resources

```bash
# List models
az ml model list \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --query "[].{Name:name, Version:version}" \
  --output table

# List endpoints
az ml online-endpoint list \
  --workspace-name "$WORKSPACE_NAME" \
  --resource-group "$RG_NAME" \
  --output table 2>/dev/null || echo "No online endpoints"
```

---

## Step 19 – Cleanup

```bash
# Delete web service
cat > delete_service.py << 'EOF'
from azureml.core import Workspace
from azureml.core.webservice import Webservice

ws = Workspace.from_config()
service = Webservice(workspace=ws, name='iris-service')
service.delete()
print("Service deleted")
EOF

python3 delete_service.py

# Delete resource group and all resources
az group delete \
  --name "$RG_NAME" \
  --yes \
  --no-wait

# Remove local files
rm -rf ml-code
rm -f *.py .azureml
rm -rf .azureml

echo "Cleanup complete"
```

---

## Summary

You created Azure Machine Learning workspace, trained a scikit-learn decision tree classifier on the iris dataset, registered the model in the workspace, deployed the model as a web service on Azure Container Instances, tested the deployed endpoint with prediction requests, and managed the complete ML lifecycle.
