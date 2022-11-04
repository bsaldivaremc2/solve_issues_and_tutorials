# How to get predictions from an exported automl model  

When you export a model from the vertex ai model registry and you want to use it for inference right away you can't. I followed the steps in their official [website](https://cloud.google.com/vertex-ai/docs/export/export-model-tabular#aiplatform_export_model_tabular_classification_sample-console) and something is missing.

Issues found:
```bash
{"error": "failed to connect to all addresses"}
```

## How to do it, all steps. Working by November 2022.
All the steps and a demonstration are available at https://youtu.be/gcQGbHnZEYQ  

- In the model registry find the model that you are interested, its version and press the button **export** button on the top right, next to view dataset, colored in blue.  

- A side window will appear requesting to select the bucket where to export your model.  
For this example the bucket folder I selected is: **gs://bsaldivar_bucket/tmp_models/** 
- below the export location a command will be automatically populated. something like:  
```bash
gsutil cp -r bsaldivar_bucket/tmp_models/ ./download_dir
```
- After you excecute that command in your computer your **usable model directory will be the folder that has a timestamp as name**. Let me explain.  If I execute the command **tree** on **./download_dir/** I will see the following:  
```bash
download_dir/
└── model-3945289613018398669
    └── tf-saved-model
        └── 2022-10-04T11:50:12.690567Z
             ├── environment.json
             ├── feature_attributions.yaml
             ├── final_model_structure.pb
             ├── instance.yaml
             ├── predict
             │   └── 001
             │       ├── assets
             │       │   ├── variable_x_vocab
             │       │   ├── .
             │       │   ├── .
             │       │   ├── .
             │       │   └── other_variable_vocab
             │       ├── assets.extra
             │       │   └── tf_serving_warmup_requests
             │       ├── saved_model.pb
             │       └── variables
             │           ├── variables.data-00000-of-00001
             │           └── variables.index
             ├── prediction_schema.yaml
             ├── tables_server_metadata.pb
             └── transformations.pb

```
The directory **2022-10-04T11:50:12.690567Z** is the directory that we will use as input for our prediction service.  
- First, you need to rename this folder with any other name that does not use **:** colon. This : interferes with the docker command line. For this example I will change the name from **2022-10-04T11:50:12.690567Z** to **2022-10-04**. Then the tree will look like this:  

```bash
download_dir/
└── model-3945289613018398669
    └── tf-saved-model
        └── 2022-10-04
             ├── environment.json
             ├── feature_attributions.yaml
             ├── final_model_structure.pb
             ├── instance.yaml
             ├── predict
             │   └── 001
             │       ├── assets
             │       │   ├── variable_x_vocab
             │       │   ├── .
             │       │   ├── .
             │       │   ├── .
             │       │   └── other_variable_vocab
             │       ├── assets.extra
             │       │   └── tf_serving_warmup_requests
             │       ├── saved_model.pb
             │       └── variables
             │           ├── variables.data-00000-of-00001
             │           └── variables.index
             ├── prediction_schema.yaml
             ├── tables_server_metadata.pb
             └── transformations.pb

```
- Other tutorials and websites will tell you to download a google container for the next step. Do not use the link from those websites. Use the link found in the **environment.json** of your model. In my case the location is **download_dir/model-3945289613018398669/tf-saved-model/2022-10-04/environment.json**  
```bash
download_dir/
└── model-3945289613018398669
    └── tf-saved-model
        └── 2022-10-04
             ├── environment.json <- This one
```
if I see in this file I will the following:  
```bash
bsaldivar@bsaldivar:~$ cd /download_dir/model-3945289613018398669/tf-saved-model/2022-10-04
bsaldivar@bsaldivar:~/download_dir/model-3945289613018398669/tf-saved-model/2022-10-04$ cat environment.json
{"container_uri": "us-docker.pkg.dev/vertex-ai/automl-tabular/prediction-server:20221014_0525", "tensorflow": "2.8.0", "struct2tensor": "0.39.0", "tensorflow-addons": "0.16.1"}
```
- the container_uri is what we need: **us-docker.pkg.dev/vertex-ai/automl-tabular/prediction-server:20221014_0525** as explained [here](https://github.com/GoogleCloudPlatform/community/issues/1556).  
- Note that I am in the directory with the modified timestamp. 
```bash
pwd
/home/bsaldivar/download_dir/model-3945289613018398669/tf-saved-model/2022-10-04
```
- Now we start the docker container that can load our exported model:  
```bash
sudo docker run -v "$(pwd)":/models/default -p 8080:8080 -it us-docker.pkg.dev/vertex-ai/automl-tabular/prediction-server:20221014_0525
```
- Finally you can follow the original website[the original website](https://cloud.google.com/vertex-ai/docs/export/export-model-tabular#aiplatform_export_model_tabular_classification_sample-console)  to submit predictions with  
```bash
curl -X POST --data @gcp_instances.json http://localhost:8080/predict
```
where gcp_instances.json is a file with one row per sample according to our trained model.