# Hints for Challenge 3

After challenge 2, we finally have a model has an accuracy of more than 99% - time to deploy it to production!
Hence, we'll now be taking the model and we'll deploy it on [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/).

Firstly, let's create a new notebook `challenge03.ipynb` for this challenge.

Our Azure Subscription must first have the providers for Azure Container Instances enabled:

```python
!az provider show -n Microsoft.ContainerInstance -o table
!az provider register -n Microsoft.ContainerInstance
```

The leading `!` tell the notebook cell to execute the code on the command line inside the Azure Notebook itself.

As before, let's connect to our Workspace:

```python
from azureml.core import Workspace, Experiment, Run
import math, random, json

ws = Workspace.from_config()
```

Since we're in a new Notebook, we need to reference our registered model from the last Notebook:

```python
from azureml.core.model import Model

model = Model(ws, name="keras-tf-mnist-model")
print(model.name, model.id, model.version, sep = '\t')
```

We need to write a short `score.py` script, that Azure ML understands for loading our model and exposing it as a webservice:

```python
%%writefile score.py
import json
import numpy as np
import os
from io import BytesIO
from keras.models import load_model
from PIL import Image
from azureml.core.model import Model
import requests

def init():
    global model
    # retreive the path to the model file using the model name
    model_path = Model.get_model_path('keras-tf-mnist-model')
    model = load_model(model_path)

def run(raw_data):
    image_url = json.loads(raw_data)['image_url']    
    image = Image.open(BytesIO(requests.get(image_url).content))
    img = np.array(image.convert('L'), dtype=np.float32).reshape(1, 28, 28, 1) / 255.0
    # make prediction
    y = model.predict(img)
    return json.dumps(y.tolist())
```

We also need to tell Azure ML which dependencies our packaged model has (similar to when we used Azure Batch AI):

```python
from azureml.core.conda_dependencies import CondaDependencies 

myenv = CondaDependencies()
myenv.add_pip_package("pynacl==1.2.1")
myenv.add_pip_package("keras==2.2.4")
myenv.add_pip_package("tensorflow==1.11.0")

with open("myenv.yml","w") as f:
    f.write(myenv.serialize_to_string())
```

Finally, we are able to configure Azure Container Instances and deploy our model to production:

```python
from azureml.core.webservice import AciWebservice, Webservice
from azureml.core.image import ContainerImage

aciconfig = AciWebservice.deploy_configuration(cpu_cores=1, 
                                               memory_gb=1, 
                                               tags={"data": "MNIST",  "method" : "keras-tf"}, 
                                               description='Predict MNIST with Keras and TensorFlow')

image_config = ContainerImage.image_configuration(execution_script = "score.py", 
                                    runtime = "python", 
                                    conda_file = "myenv.yml")

service = Webservice.deploy_from_model(name = "keras-tf-mnist-service",
                                       deployment_config = aciconfig,
                                       models = [model],
                                       image_config = image_config,
                                       workspace = ws)

service.wait_for_deployment(show_output = True)
```

The first run should take around 10 minutes, as again, the Docker image needs to be build.

In our Workspace, we can check the `Images` tab:

![alt text](../images/03-docker_creating.png "Our production image is being created")

Shortly after, we should also see our ACI service coming up under the `Deployments` tab:

![alt text](../images/03-aci_creating.png "Our ACI service is starting")

Lastly, we can print out the service URL:

```python
print(service.state)
print(service.scoring_uri)
```

Let's try to make a request:

```python
import requests
import json

headers = {'Content-Type':'application/json'}
data = '{"image_url": "https://dpk18.blob.core.windows.net/test/4.png"}'

resp = requests.post(service.scoring_uri, data=data, headers=headers)
print("Prediction Results:", resp.text)
```