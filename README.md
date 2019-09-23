[![Build Status](https://defradev.visualstudio.com/DEFRA_FutureFarming/_apis/build/status/defra-ffc-demo-user-service?branchName=master)](https://defradev.visualstudio.com/DEFRA_FutureFarming/_build/latest?definitionId=594&branchName=master)
[![Known Vulnerabilities](https://snyk.io//test/github/DEFRA/ffc-demo-user-service/badge.svg?targetFile=package.json)](https://snyk.io//test/github/DEFRA/ffc-demo-user-service?targetFile=package.json)

# FFC Demo User Service

Digital service mock to claim public money in the event property subsides into mine shaft.  The user service receives user data and if it doesn’t already exist saves it in a Postgresql database table.

# Prerequisites

Either:
- Docker
- Docker Compose

Or:
- Kubernetes
- Helm

Or:
- Node 10
- PostgreSQL database

# Environment variables

The following environment variables are required by the application container. Values for development are set in the Docker Compose configuration. Default values for production-like deployments are set in the Helm chart and may be overridden by build and release pipelines.

| Name              | Description       | Required | Default     | Valid                     | Notes |
|-------------------|-------------------|:--------:|-------------|---------------------------|-------|
| NODE_ENV          | Node environment  | no       | development |development,test,production|       |
| PORT              | Port number       | no       | 3002        |                           |       |
| POSTGRES_USERNAME | Postgres username | yes      |             |                           |       |
| POSTGRES_PASSWORD | Postgres password | yes      |             |                           |       |
| POSTGRES_DB       | Postgres database | yes      |             |                           |       |
| POSTGRES_HOST     | Postgres host     | yes      |             |                           |       |
| POSTGRES_PORT     | Postgres port     | yes      |             |                           |       |

# How to run tests

A convenience script is provided to run automated tests in a containerised environment:

```
scripts/test
```

This runs tests via a `docker-compose run` command. If tests complete successfully, all containers, networks and volumes are cleaned up before the script exits. If there is an error or any tests fail, the associated Docker resources will be left available for inspection.

Alternatively, the same tests may be run locally via npm:

```
npm run test
```

Running the tests locally requires a Postgres database for integration tests, and the following environment variables:

- `POSTGRES_USERNAME`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `POSTGRES_HOST`

# Running the application

The application is designed to run in containerised environments: Docker Compose for development; Kubernetes for production.

A Helm chart is provided for deployment to Kubernetes and scripts are provided for local development and testing.

## Build container image

Container images are built using Docker Compose and the same image may be run in either Docker Compose or Kubernetes.

The [`build`](./scripts/build) script is essentially a shortcut and will pass any arguments through to the `docker-compose build` command.

```
# Build images using default Docker behaviour
scripts/build

# Build images without using the Docker cache
scripts/build --no-cache
```

## Run as an isolated service

To test this service in isolation, use the provided scripts to start and stop a local instance. This relies on Docker Compose and will run direct dependencies, such as message queues and databases, as additional containers. Arguments given to the [`start`](./scripts/start) script will be passed through to the `docker-compose up` command.

```
# Start the service and attach to running containers (press `ctrl + c` to quit)
scripts/start

# Start the service without attaching to containers
scripts/start --detach

# Send a sample request to the /register endpoint
curl -i --header "Content-Type: application/json" --request POST --data '{ "email": "test@email.com" }' http://localhost:3002/register

# Stop the service and remove Docker volumes and networks created by the start script
scripts/stop
```

## Connect to sibling services

To test this service in combination with other parts of the FFC demo application, it is necessary to connect each service to an external Docker network and shared dependencies, such as message queues. Start the shared dependencies from the [`ffc-demo-development`](https://github.com/DEFRA/ffc-demo-development) repository and then use the `connected-` [`scripts`](./scripts/) to start this service. Follow instructions in other repositories to connect each service to the shared dependencies and network.

```
# Start the service
script/connected-start

# Stop the service
script/connected-stop
```

## Deploy to Kubernetes

For production deployments, a helm chart is included in the `.\helm` folder. This service connects to an AMQP 1.0 message broker, using credentials defined in [values.yaml](./helm/values.yaml), which must be made available prior to deployment.

Scripts are provided to test the Helm chart by deploying the service, along with an appropriate message broker, into the current Helm/Kubernetes context.

```
# Deploy to current Kubernetes context
scripts/helm/install

# Remove from current Kubernetes context
scripts/helm/delete
```

### Accessing the pod

The ffc-demo-user-service is not exposed via an endpoint within Kubernetes.

The deployment may be accessed by forwarding a port from a pod.
First find the name of the pod by querying the namespace, i.e.

`kubectl get pods --namespace ffc-demo-user-service-pr2`

This will list the full name of all the pods in the namespace. Forward the pods exposed port 3002
to a local port using the name returned from the previous command, i.e.

`kubectl port-forward --namespace ffc-demo-user-service-pr2 ffc-demo-user-service-8b666f545-g477t  3002:3002`

Once the port is forwarded a tool such as [Postman](https://www.getpostman.com/) can be used to access the API at http://localhost:3002/register.

Sample valid JSON that can be posted is:

```
{
  "email": "test@email.com"
}
```

Alternatively, curl can be used to send a request to the end point:

```
curl -i --header "Content-Type: application/json" --request POST --data '{ "email": "test@email.com" }' http://localhost:3002/register
```

# Database migrations

Running the npm script

```
npm run migrate
```

Will run all the migration scripts to bring the database up to date. This script is run as part of the test suite, in order to create the database tables that are required for the tests to run.

The migrate script uses the `sequelize-cli` module to run any update scripts that are outstanding. `sequelize` uses the `umzug` module internally. The `dbversion` code uses this module to see if any migrations are outstanding when the service starts. The readiness probe also uses the `dbversion` code to check if the latest database migration is known to it. If it is not (if for example another version has performed database updates) then the readiness monitor will return an error, so that traffic is not routed to this instance.

This check is not run before every call deliberately. This would pose a performance issue. If the service was performing hundreds of requests per second, checking before every request would impose a penalty for very little gain. It would also be pointless, as errors should be dealt with properly at the time they occur, rather than trying to anticipate every one. There is also a race condition here, as even if the check was done before a request was handled it doesn't guarantee that the database would still be in the same state when the request completes.

## Database migrations under kubernetes

```
TBC - Currently database migrations are only done as part of the PR and are not done during any part of the kubernetes initialisation.
```

# Build pipeline

The [azure-pipelines.yaml](azure-pipelines.yaml) performs the following tasks:
- Runs unit tests
- Publishes test result
- Pushes containers to the registry tagged with the PR number or release version
- Deletes PR deployments, containers, and namepace upon merge

Builds will be deployed into a namespace with the format `ffc-demo-user-service-{identifier}` where `{identifier}` is either the release version, the PR number, or the branch name.

A detailed description on the build pipeline and PR work flow is available in the [Defra Confluence page](https://eaflood.atlassian.net/wiki/spaces/FFCPD/pages/1281359920/Build+Pipeline+and+PR+Workflow)

## Testing a pull request

A PR can be tested by reconfiguring the user service to use the URL of the PR rather than the current release in the development cluster. Create a `patch.yaml` file containing the desired URL:

```
apiVersion: extensions/v1beta1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - env:
        - name: FFC_DEMO_USER_SERVICE
          value: http://ffc-demo-user-service.ffc-demo-user-service-pr2
        name: ffc-demo-user-service
```

then apply the patch:

`kubectl patch deployment --namespace default ffc-demo-user-service --patch "$(cat patch.yaml)"`

Once tested the patch can be rolled back, i.e.

`kubectl rollout undo --namespace default deployment/ffc-demo-user-service`
