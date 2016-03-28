# Dev,Create,Deploy and Test API Gateway + Python Lambda Functions

This repository provides a framework for writing, packaging, and
deploying lambda functions to AWS.

You can also auto create or update your API on AWS API Gateway. You can also test connectivity to it and even call AWS Cognito to obtain an IdentityID.

You can simluate a Mobile App behavior and play the entire flow locally:

Cognito -> Connect to API -> Call Lambda function -> Parse result file

## Cognito

If your API is not public and is secured by IAM Roles, you will need the proper credentials to connect to it. Cognito allows your clients to obtain an identity and a token they can obtain temporary Credentials.

Those credentials are set by AWS IAM roles and grant access to AWS services. In our case, the services we need access to are: AWS API Gateway and AWS Lambda (if your lambda function is configured to pass through user credentials)

## API Gateway & Swagger

You should define your API in a description file. Swagger is a tool that provides a YAML format for describing your API. It also generates documentation for you and an online service: http://swaggerhub.com

AWS supports swagger files in order to create your API. There is an example of a yaml file in the "swagger" folder that respects the Swagger/AWS format and requirements.

Using the `aws-apigateway-importer` program provided by AWS and added as a submodule here, you can automatically create, update and deploy your API in AWS. No need for console anymore.

The YAML file is authoriative and always accurratly describes your API. 

## Setup your env

* Python 2.7
* Pip 6+
* aws-cli 1.9+

Install AWS CLI tool:
http://docs.aws.amazon.com/cli/latest/userguide/installing.html#install-with-pip

Configure credentials:
http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html

If you have some issues with dependencies, consider using VirtualEnv for Python.

## Writing Functions

Function code goes in the `src/` directory. Each function must be in a
sub-directory named after the function. For example, if the Lambda
function name is "signin", the code goes in the `src/signin`
directory.

*Note:* Make sure you put an `__init__.py` in your folder and any
 subfolders. Check existing functions.

### Entry Point

The entrypoint for each function should be in an `index.py` file,
inside of which should be a function named `handler`. See the AWS
Lambda documentation for more details on how to write the handler.

    # src/<function>/index.py
	def handler(event, context):
		...

### Third-party Libraries

For a third-party library, use the `requirements.txt` file as you
would any Python package.  See the Python pip documentation for more
details:
https://pip.readthedocs.org/en/stable/user_guide/#requirements-files

### Internal Libraries

If you need to write a common library that will be shared among
multiple Lambda functions, but is not independent enough to warrant
its own package, you can use the `lib` directory.

Consider the `lib` directory to be a part of the `PYTHON_PATH`. This
means any module or sub-modules in that directory will be available in
any function. (Behind the scenes, the contents of the `lib` directory
are copied verbatim alongside the Lambda function source files.)

## Makefile

All is done through the Makefile.

```
Run a function:        make run/FUNCTION [EVENT=filename]
Run all tests:         make test
Run a specific test:   make test/TEST
----------------------------------------------------------
Create AWS function:   make create/FUNCTION
Package all functions: make dist
Package a function:    make dist/FUNCTION
Deploy all functions:  make deploy [ENV=prod] - Default ENV=dev.
Deploy a function:     make deploy/FUNCTION [ENV=prod]
Setup environment:     make env [ENV=environment] - Downloads the config file from S3. (edit Makefile)
Set function MEM size: make setmem/FUNCTION SIZE=[size] - Set/Update a function memory size
----------------------------------------------------------
Deploy an API to AWS:  make api VERS=<version> [UPDATE=<api_id>] [STAGE=<stage_name>] [CREATE=1]
       	      	       The Makefile expects a file in the `swagger` folder as follow: swagger/api-${VERS}.yaml
                       We process this file and perfom replacement of possible %% variables in the file.
		       Edit Makefile to process your custom variables.
                       You can decide to deploy the API straight to AWS API Gateway and deploy it.
                       If UPDATE is provided: It will update the API directly in AWS using the ID provided
                       If STAGE is provided: It will also deploy the API once updated
                       If CREATE is provided: It will create the API
----------------------------------------------------------
Connect to a live API: make connect METHOD=<method> ENDPOINT=</path/{id}> [QUERY=<foo=bar&bar=foo>] [PAYLOAD=<filepath>] [NOAUTH=1]
                       METHOD: GET, POST, PUT, PATCH, DELETE, etc
                       ENDPOINT: refers to the URL path after `hostname/stage/`. Example: `/users`, `/videos`, etc 
                       QUERY: refers to the querystring passed to the URL. Standard querystring. Example: `id=2134&format=json`
		       
                       You can provide payload with STDIN. Just type your input JSON data, then hit 'ctrl-d' to exit input and send the input data.

		       NOAUTH forces a new Cognito Identity and ignore the .identity file.

		       The .identity file caches your Cognito identity locally. We create it at the first call and cache it. When you test your lambda functions locally, we pass the identity to the functions in the `context` variable. We Mock what you would receive from AWS.

                       Examples:
                       POST /signin: Sign the user in
                       `$> make connect ENDPOINT=/signin METHOD=POST PAYLOAD=tests/data/signin.json`

                       GET /assets/{asset_id}: Get an asset
                       `$> make connect ENDPOINT=/assets/0e0d1e42ad20443c METHOD=GET`
```

### Running Functions Locally

Before building and deploying, you may want to test your Lambda code
first. In order to simulate the Lambda environment, a script is
provided that will execute your function as if it was in AWS Lambda.

	usage: make run/FUNCTION [VERBOSE=1] [EVENT=filename]

	Run a Lambda function locally.

	optional arguments:
	  VERBOSE=1       Sets verbose output for the function
	  EVENT=filename  Load an event from a file rather than stdin. This file contain the input you want to send to your function.

The `make run/%` script will take a JSON event from a standard input
(or a file if you specify), and execute the function you specify.

### Writing Unit Tests

For automated testing, you can write unit tests using Python's
`unittest` module.

* All test cases must be in a file called `test*.py` in the `tests`
  directory. Any example is `testModule.py`.
* Each file can contain any number of test cases, but each must
  inherit from `unittest.TestCase`.
* In tests, you can import the handler for a Lambda function with:
  `import src.<MODULE_NAME>.index`, and then using `.handler`.

All the unit tests can be run using `make test`. It auto-discovers all
test cases that follow the above rules. To run a specific test, just
run `make test/<TEST>`, replacing `<TEST>` with the name of the test
(i.e., the part of the test's filename after "test"; as an example,
the file "testUpload.py" has the name "upload").

### Creating

If the function is new you must create it in the system.

        make create/<function>

This will create the function in lambda. You can then update it using
"deploy" below.

### Building

Once your function is written, it can be built into a ZIP file.

	make dist/<function>.zip

Run the above command to build a ZIP archive for "function". It will
automatically get the necessary Python packages.

Note: If you didn't get the Python modules. Just run first:

        make clean

You can also run `make all` or `make dist` to build all of the
functions.

### Deploying

Finally, you can deploy a function by running:

	make deploy/<function> [ENV=prod]

This will build the ZIP file, upload it to S3, and finally update the
function code on Lambda.

As with building, you can call `make deploy` to simultaneously deploy
all Lambda functions. (Use this carefully.)

You want to specify ENV=prod if you want to pull the .env file from prod S3 and not the default DEV.
If you do so, make sure you have the AWS credentials setup correctly locally.

### Connecting

If your API is secure, you can't test it in the AWS console nor in the browser anymore. You need to authenticate.

If you use Cognito, you can connect to your API by using this framework:

 	make connect ENDPOINT=/my_endpoint METHOD=GET QUERY="parm1=val1"

This will call your API and display the resulting data. We expect JSON.

## Environment & Secrets

In order for your Lambda functions to get secrets and environment variables, we provide a mechanism in the Makefile that downloads a file from S3 and transforms it as a env.py file put in `/lib`. This file can then be `import` in your functions and will be added in the resulting ZIP file before being sent out to AWS.

### Details

If you want to use environment variables, just put the line below at the
top of your index.py file.

	from lib import env

To refresh the local .env file, delete it, and

   	make .env [ENV=prod]

We could download from S3 at runtime, but it slows down the Lambda function, and you pay for that.

Secrets such as the following can be added to this file in S3:
  - Environment variables
  - Secret API keys
  - Anything else that is external and dynamic

Note: The .env file will be stored locally from the environment file downloaded from S3.

Note: Edit the `.env` rule in the Makefile to download your configuration file from the S3 location you want.
