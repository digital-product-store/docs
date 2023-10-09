# docs

currently, only 2 services are implemented as ready-to-run and running on the mesh: user service and product service. both are http services. they have their pipeline basically runs lint and format checkers, builds docker image, push ECR and then deploy latest versions to EKS by helm. For keeping implementations minimal, pipelines deploy applications by `helm upgrade` instead of any proper CD tool like argocd or flux.

you can find the helm chart repository of the applications here: https://github.com/digital-product-store/helm_charts and you can see the charts' sources here: https://github.com/digital-product-store/helm_charts_src (templates are so basic and not parameterized well.)

currently there are tree packages total: `user`, `product` and `reqauth`. the user and product packages are packages of the applications as their names suggest, `reqauth` is the package defines `RequestAuthentication` manifest for the istio mesh to leverage JWT authentication to istio instead of being performed by services' themselves. thanks to istio, `product` service does not include any authentication / authorization logic even it has some `admin` endpoints that they need to be open only for permitted users. authn and authz processes are performed by istio, it validates given JWT, then it checks user' role if it is `admin` or `user` depending on the endpoint the user try to reach and then it extracts the `user_id` from JWT and passes to services as identifier of the user.

each service are implemented as compatible with openapi3 specs, all of them are exposing an endpoint that you can reach the specification, but these endpoints are not allowed to access in the mesh from public, you need to run applications on your local. for that purpose, you can use `docker-compose.yml` then visit their specification endpoints, for user service it is `/_openapi3.json` and it is `/docs` for `product` service.

you can visit the infra repository to see how terraform code creates the environment on AWS: https://github.com/digital-product-store/infra

## services

the `user` service uses an `in-memory` database and exposes one endpoint to public: `POST /api/v1/auth` which is used for authenticate a user and get a token. this service has no dependency.

the `product` service has a database (postgresql) in RDS and an S3 bucket dependencies. to see how dependencies are created and how service account policy is defined visit `infra` repository (`rds.tf`, `s3.tf`, `product_iam.tf`). the `product` services exposes 3 different endpoints: `POST /admin/api/v1/upload`, `POST /admin/api/v1/book` and `GET /api/v1/books`. the endpoints starts with `/admin` requires an bearer token and only accessible if your user has `admin` role, other is publicly accessible.

you can import the environment and request collections to your postman application in order to see details and make requests.

you can try to reach `/admin` endpoints of the `product` service by using a token you can get from the `user` service. if you check the **in-memory user database** [here](https://github.com/digital-product-store/user/blob/main/cmd/user/user.go#L40) you can see some user holds `admin` in their `Roles` attirbute. In order to be able to access `/admin` endpoints of the `product` service, you need to use a user's credentials (`Username` and `Password` attributes) who holds `admin` rights, otherwise you will get error response *(istio returns 404 default, but it is configurable.)*.

## new version release

each service has a `publish` pipeline runs every commit on the `main` branch, so if you want to deploy a new version you can basically follow these steps below:
- go to the user service repository
- add a new user into `in-memory` database here: https://github.com/digital-product-store/user/blob/main/cmd/user/user.go#L40
- create a PR and then merge
- check the deployment: `kubectl describe deployments user-deployment`

## TODO

- implement other services: `order`, `exchange`, `payment`
- implement and demonstrate istio examples for new services: circuit breaker, retry, rate limiter etc.
- implement some `private` endpoints for services that are only accessible inside of mesh. make services talking each other.
- implement **contract testing** to make sure new releases are not breaking anything and still compatible with other services in prod. see [pact.io](https://pact.io)
- observability: log centralization (elk), distributed tracing (elk), structured logging, mesh visualization (kiali.io)
- argocd for proper deployments.
