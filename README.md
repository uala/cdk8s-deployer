# CDK8S Deployer

This tool can be used as a step in a CI/CD pipeline for deploy one or more applications from a IaC repository based on a YAML config file.


Plugin in action:

![Execution](/example-deployer.png)

## Usage

In order to use this tool, some environment variables are required:
* `GIT_IAC_REPO`: This is the most important settings, it contains the IaC repository that you want to deploy
* `GIT_IAC_BRANCH`: By default the tool will use the default branch (master/main). You can set a different branch here.
* `GIT_IAC_TOKEN`: A github token authorized to access in read/write to the Iac Repo
* `DEPLOY_ENVIRONMENTS`: A comma list of environments you want to deploy (see also conf file)
* `DEPLOY_AGE_KEYS`: A comma list of age keys used to decrypt secrets (see also conf file)
* `DEPLOY_CONF_FILE`: relative path name of the config file (ex. `conf.yaml`)
* `DEPLOY_DRY_RUN`: Simulate the k8s deploy with a dry run. Available values are `client` (or `true`) and `server`. Attention: with Rancher a dry run will create anyway the project and namespaces, due to a limitation on RBAC
* `DEPLOY_ASK_CONFIRM`: Ask for a user confirm before deploy (ex. `true`)

You can run it with docker with:
```
docker run -it --env-file $PWD/.env -v $PWD/conf.yaml:/usr/src/app/conf.yaml uala/cdk8s-deployer
```

### Configuration file
This tool can be run also with a conf file (using `DEPLOY_CONF_FILE` env).
This is an example of the file:
```yaml
deploy_environments:
- develop:
  - beAdmin
  - beAuth
  - beMain
  - beBrands
  - beGraph
env_vars:
  - RANCHER_URL: "https://your.rancherinstance.com"
  - RANCHER_ACCESS_KEY_ENV: "token-rancher"
  - RANCHER_SECRET_KEY_ENV: "rancher-secret"
age_keys:
  - AGE-SECRET-KEY-123
```
This file can be used in substitution or in combination with env variables.
This also support a set of enviornment variables that will be automatically injected.
This file can be automatically generated by [cdk8s-image-updater](https://github.com/uala/cdk8s-image-updater)

### Cluster conf file
This tool requires a configuration file on the IaC Repository where it can find the definition of clusters where it will operate.
This is a YAML (with ERB support) file stored in the root of your repository and with a syntax like `clusters*.yaml`.
Multiple files are supported.

Example `clusters.yaml`:
```yaml
clusters:
  - name: aws-test-version
    settings:
      rancher_url: <%= ENV['RANCHER_URL'] %>
      rancher_access_key: <%= ENV['RANCHER_ACCESS_KEY_ENV'] %>
      rancher_secret_key: <%= ENV['RANCHER_SECRET_KEY_ENV'] %>
    environments:
      - name: "develop"
        settings:
          rancher_project: "test-iac"
      - name: "production"
        settings:
          secret: clusters/production.enc.yaml
          rancher_project: "test-iac"
          rancher_access_key: <%= ENV['RANCHER_ACCESS_PROD_KEY_ENV'] %>
          rancher_secret_key: <%= ENV['RANCHER_SECRET_PROD_KEY_ENV'] %>
```

In the example above, the tool expects there are 2 environments in these paths:
```
applications/environments/develop
applications/environments/production
```

#### clusters file reference

* `name`: It contains the exactly name of the cluster
* `environments`: A list of environments the cluster should contain, based on the structure of the Iac Repo
* `settings`: Useful settings for the tool, like rancher credentials and rancher project. This section can exist at cluster level or at application level, missing informations will be merged with cluster ones.

In setting sections the tool expects to find how can access the cluster and how to deploy.
In the example above, the tool will use Rancher to connect to the cluster and deploy environments to the specific Rancher Project defined.

In the setting section you can also specify the path of a `secret` where to find the auth method.
The secret should be encrypted with `sops` and should have the following structure:
```yaml
name: cluster-name
data:
    KUBE_CONFIG: |-
        apiVersion: v1
        kind: Config
        clusters:
        - name: "your-cluster-kubeconfig"
        ...
    IAM_USER:
        AWS_ACCESS_KEY_ID: YOUR_IAM_ACCESS_KEY
        AWS_SECRET_ACCESS_KEY: YOUR_IAM_SECRET_KEY
        AWS_DEFAULT_REGION: eu-west-1
    RANCHER:
        SERVER_URL: https://your.rancherinstance.com
        ACCESS_KEY: token-rancher
        SECRET_KEY: rancher-secret
```
As you can see the tool supports 3 different auth methods in the secret:
* Plain kubeconfig
* AWS IAM User with access to eks and to the cluster (a kubeconfig will be generated in realtime)
* Rancher Credentials

The tool will use only one of the three methods defined in the above order, no fallback is supported atm.

## Development

After checking out the repo, run `bundle install` to install dependencies.

Run with `ruby deployer.rb`


## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/uala/cdk8s-deployer

## License

Iac Image Updater is released under the [MIT License](https://opensource.org/licenses/MIT).
