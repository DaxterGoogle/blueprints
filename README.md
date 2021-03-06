# tldr
Basic Config Controller setup managing resources under a GCP folder 

# Prereqs

* GCP folder where KCC resources will live
* Project under the folder where the config controller cluster will live
* nomos, kubectl, and kpt CLI's

`gcloud components install pkg` 

# Cluster setup
```

#Folder_ID is the preexisting folder where all KCC resources live under
#PROJECT_ID is the preexisting project where the config controller cluster will live. The project should live inside the  folder.
export FOLDER_ID=Change_Me_FOLDER_ID
export PROJECT_ID=Change_Me_PROJECT_ID
export CONFIG_CONTROLLER_NAME=config-controller-1


gcloud config set project ${PROJECT_ID}
mkdir  ${PROJECT_ID}
cd  ${PROJECT_ID}

gcloud services enable krmapihosting.googleapis.com container.googleapis.com

gcloud anthos config controller create ${CONFIG_CONTROLLER_NAME} \
  --location=us-central1
  gcloud anthos config controller get-credentials ${CONFIG_CONTROLLER_NAME} \
  --location us-central1

export SA_EMAIL="$(kubectl get ConfigConnectorContext -n config-control \
  -o jsonpath='{.items[0].spec.googleServiceAccount}' 2> /dev/null)"

gcloud resource-manager folders add-iam-policy-binding "${FOLDER_ID}" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/owner"

gcloud resource-manager folders add-iam-policy-binding "${FOLDER_ID}" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/resourcemanager.folderAdmin"

gcloud services enable cloudresourcemanager.googleapis.com
```
# Gitops setup 

```
kpt pkg get https://github.com/DaxterGoogle/blueprints.git/catalog/gitops@main
# SSH -> kpt pkg get ssh://git@github.com/DaxterGoogle/blueprints.git/catalog/gitops@main

#get project # for setters.yaml
gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)'

# Fill out project-id and project-number gitops/setters.yaml
# This is the project where the config controller cluster lives

kpt fn render gitops/
kpt live init gitops/ --namespace config-control
kpt live apply gitops/ --output table

gcloud source repos clone source-repo
mv gitops source-repo/
cd source-repo/
git config init.defaultBranch main
git add gitops/
git commit -m "Add GitOps blueprint"
git push

kubectl apply -f gitops/configsync/root-sync.yaml
```

# Landing zone setup

```
kpt pkg get https://github.com/DaxterGoogle/blueprints.git/catalog/landing-zone-lite@main
# SSH -> kpt pkg get ssh://git@github.com/DaxterGoogle/blueprints.git/
# Fill out folder-id and management-project-id landing-zone-lite/setters.yaml
git add landing-zone-lite/
git commit -m "Add landing zone"
git push

```
# Billing permisions for project creation 
The [projects namespace](https://github.com/DaxterGoogle/blueprints/blob/main/catalog/landing-zone-lite/namespaces/projects.yaml#L132)  [ConfigConnectorContext](https://github.com/DaxterGoogle/blueprints/blob/main/catalog/landing-zone-lite/namespaces/projects.yaml#L138) [GCP service account](https://github.com/DaxterGoogle/blueprints/blob/main/catalog/landing-zone-lite/namespaces/projects.yaml#L15) requires the billing account user role on the billing account used to create projects.

A user with billing admin permisions on the  billing account will need to provide this 

gcloud beta billing accounts add-iam-policy-binding billing-account-id --member=serviceAccount:projects-sa@${PROJECT_ID}.iam.gserviceaccount.com