# Clustering Systems

## Running the clustering on OpenShift with mlflow
First, we need to get the application code on OpenShift. Once you have the code on openshift, you can have multiple instances of your experiment running with different parameters.

To run this application on openshift, we need to create a container image for it. In other words, this means that we need to download the code that we want to run on openshift. To do this a image build template should already be available in your openshift namespace.

To download the source code for our experiment to openshift, use the following command:
```

oc process mlflow-experiment-bc --param APPLICATION_NAME=your-application-name --param GIT_URI=https://github.com/ManageIQ/aiops-insights-clustering.git --param APP_FILE=app.py | oc create -f -

```
- Here, the APPLICATION_NAME parameter is the name of the container image that will be built and is where your code will be downloaded, this is the image name that you will use to run your experiments.
- You can set GIT_URI to the link for your repository to create a container image for your application.
- You can also change the APP_FILE env variable to your app file name.
 
If the image build process has started you should see some output like this:

```
    imagestream.image.openshift.io "my-mlflow-experiment" created
    buildconfig.build.openshift.io "my-mlflow-experiment" created
    
```

To see if the build process has finished, run the following command:
```
    oc logs bc/your-application-name
    
```
If you see something like:
```
    Pushed 5/10 layers, 50% complete
    Pushed 6/10 layers, 60% complete
    Pushed 7/10 layers, 70% complete
    Pushed 8/10 layers, 80% complete
    Pushed 9/10 layers, 90% complete
    Pushed 10/10 layers, 100% complete
    Push successful
```
at the end of the logs, the build is complete.

## Development workflow

Copy .env file and adjust variables

```
cp .env.example .env
```

After the image is built, we can use it to run the clustering on OpenShift.

Edit the Makefile to set the APP_IMAGE_URI to your application image as follows:

```
  oc new-app mlflow-experiment-job --param APP_IMAGE_URI=your-application-image-name\
```

To run the clustering:
```
  make oc_mlflow_job
```

Look at the output of the job

```
❯ oc logs systems-clustering-job-spah -f
---> Running application from Python script (app.py) ...
app.py:50: FutureWarning: Method .as_matrix will be removed in a future version. Use .values instead.
  np_rules = pd_rules.as_matrix()
0        1
1        4
2        1
        ..
40160    3
40161    1
Name: cluster, Length: 40162, dtype: int32
```

## Tests

```
make test
```

## Local build

If you would like to deploy the clustering service locally, you can build the container using [S2I](https://github.com/openshift/source-to-image)

```
❯ s2i build -c . centos/python-36-centos7 aicoe-insights-clustering
```

For convenience you can store your desired environment variables in a separate file

```
❯ cat << EOT >> env.list
FLASK_ENV=development
CEPH_KEY=...
CEPH_SECRET=...
CEPH_ENDPOINT=...
EOT
```

And then run it as a Docker container

```
❯ docker run --env-file env.list  -p 8080:8080 -it aicoe-insights-clustering
```

# Data storage

Currently we support 3 types of data storage. Clustering service selects the proper one based on environment variables.

- Ceph (use `CEPH_KEY`, `CEPH_SECRET` and `CEPH_SECRET`)
- AWS S3 (use `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`)
- Local (neither variable from above is set)


**Please note that AWS S3 access is not intended to be used in development since it may touch sensitive production data.**
**For development purposes please use the local storage as described later.**


## Fetching a local copy of data from AWS

Install and configure [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

```
❯ pip install awscli --user
...
❯ aws configure --profile insights
AWS Access Key ID [None]: <YOUR_ACCESS_KEY>
AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
Default region name [None]: <leave_blank>
Default output format [None]: <optional [json|text|table]>
```

Sync data locally (replace `<YYYY-MM-DD>` with an existing date). Use `DH-DEV-INSIGHTS` bucket which contains nonsensitive data only, please.

```
❯ # List available dates
❯ aws s3 ls --profile insights \
            --endpoint-url=http://storage-016.infra.prod.upshift.eng.rdu2.redhat.com:8080/ \
            s3://DH-DEV-INSIGHTS/
PRE 2018-02-28/
PRE 2018-03-01/
PRE 2018-03-02/
PRE 2018-03-03/
...

❯ # Sync to ./data
❯ aws s3 sync --profile insights \
              --endpoint-url=http://storage-016.infra.prod.upshift.eng.rdu2.redhat.com:8080/ \
              s3://DH-DEV-INSIGHTS/<YYYY-MM-DD>/rule_data ./data
```
