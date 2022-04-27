# blueprints

# Cluster setup
```
export PROJECT_ID=PROJECT_ID
export CONFIG_CONTROLLER_NAME=config-controller-1
export FOLDER_ID=FOLDER_ID


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

kpt pkg get ssh://git@github.com/DaxterGoogle/blueprints.git/catalog/gitops@main
#get project #
gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)'
# Fill out gitops/setters.yaml
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

kpt pkg get ssh://git@github.com/DaxterGoogle/blueprints.git/catalog/landing-zone-lite@main
# Fill out landing-zone-lite/setters.yaml
git add landing-zone-lite/
git commit -m "Add landing zone"
git push

```
