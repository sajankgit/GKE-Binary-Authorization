# GKE Binary Authorization


As a DevSecOps Engineer, I wish application images that I trust should only be deployed in my infrastructure. My cluster should reject all other un-trusted images.

Google Kubernetes Engine provides a feature **Binary Authorization** which can help us achieve the above goal.

Binary Authorization is a deploy-time security control that ensures only trusted container images are deployed on Google Kubernetes Engine (GKE) or Cloud Run. It is a service on Google Cloud that provides centralized software supply-chain security for applications that run on Google Kubernetes Engine (GKE).


![image](https://user-images.githubusercontent.com/37524392/157366127-92cfc004-6ec4-4faf-8c90-26ca6620e0c3.png)

> Image Credits: https://codelabs.developers.google.com

With Binary Authorization, you require images to be signed by trusted authorities during the development process and then enforce signature validation when deploying.

By enforcing validation, you can gain tighter control over your container environment by ensuring only verified images are integrated into the build-and-release process.

Ref: https://cloud.google.com/binary-authorization/docs



In this blog, I am going to implement Binary authorization in a GKE cluster. The aim is to allow only images that is built using Circle CI. (Assuming Circle CI checks have been passed before pushing the image to the registry)

Before starting, Let’s understand some keywords that will be used while implementing GKE binary authorisation.

- **Binary Authorization** is a deploy time security service provided by Google that ensures that only trusted containers are deployed in our GKE cluster. It uses a policy driven model that allows us to configure security policies. Behind the scenes, this service talks to the Container Analysis service.

- **Container Analysis** is an API that is used to store trusted metadata about our software artifacts and is used during the Binary Authorization process

- **Attestor** is a person or process that attests to the authenticity of the image

- **Note** is a piece of metadata in Container Analysis storage that is associated with an Attestor

- **Attestation** is a statement from the Attestor that an image is ready to be deployed. In our case we will use an attestation that refers to the signing of our image


## Enable Binary Authorisation

- Go to the Security page at Google Cloud Console.

- Enable the Binary Authorization API if not

- Go to the Kubernetes Engine page at Google Cloud Console.

- Select the cluster and click EDIT.

- Set Binary Authorization to Enabled.

- Click SAVE.


![image](https://user-images.githubusercontent.com/37524392/157366778-97ec0d23-c9cd-46af-858c-410173af7a7f.png)


You can also do it through gcloud command.

```
gcloud container clusters create \
    --enable-binauthz \
    --zone <zone> \
    <cluster-name>
```



## Create Attestor


- Create a PKIX key pair
  ```
  ## Create Key ##
  openssl ecparam -genkey -name prime256v1 -noout -out ${PRIVATE_KEY_FILE}
  ## extract the public key ##
  openssl ec -in ${PRIVATE_KEY_FILE} -pubout -out ${PUBLIC_KEY_FILE}
  ```

- Go to the Binary Authorization page for the attestor project.

- In the Attestors tab, click Create.

- Click Create New Attestor.

- In Attestor Name, enter a name for the attestor.

- Select Automatically Generate a Container Analysis Note to create a new [note](https://cloud.google.com/binary-authorization/docs/key-concepts#analysis-notes).

- Add the public key to the attestor.

![image](https://user-images.githubusercontent.com/37524392/157367070-813da0df-a8f1-4807-88f1-e5be93352d80.png)

- Create Policy so that Images Attested by above attestor are only deployed.

![image](https://user-images.githubusercontent.com/37524392/157367197-a7c24ffa-0cb2-4b6d-922e-06e5fda4d480.png)

Please note that these steps can be done programatically using `gcloud` command. ref: [https://cloud.google.com/binary-authorization/docs](https://cloud.google.com/binary-authorization/docs)


## Attesting An Image

- Create payload Json with the artifact URL.

```
gcloud container binauthz create-signature-payload \
  --artifact-url="<artifact-url>/<image-path>@<image-digest" > generated_payload.json
```

- Sign the payload json with the private key created

  `openssl dgst -sha256 -sign private.key generated_payload.json > ec_signature`
  
- Get public key ID from the attestor to create attestations.
  
  `gcloud container binauthz attestors describe <attester-name> \
  --format='value(userOwnedGrafeasNote.publicKeys[0].id)' --project <project-name>`

- Create the attestation
  
  ```
  gcloud container binauthz attestations create \
    --project="<project-name>" \
    --artifact-url="<artifact-url>/<image-path>@<image-digest>" \
    --attestor="projects/<project-name>/attestors/<attestor-name>" \
    --signature-file="ec_signature" \
    --public-key-id="<PUBLIC_KEY_ID>" \
    --validate
  ```


**Note**: We attest the images with its Digest instead of tag. So in deployment file we should give image digest in image name.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <name>
  template:
    metadata:
      labels:
        app: <name>
    spec:
      containers:
      - name: <name>
        image: **<artifact-url>/<image-path>@<image-digest>**
```

