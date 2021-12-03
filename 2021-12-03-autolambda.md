---
title: "Automating Lambda Deployment With Travis"
created_at: Fri Dec 3 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2012-12-03-autolambda
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![TCI-Graphics for AdsBlogs (4)](https://user-images.githubusercontent.com/20936398/144677834-c06d74ca-d6d3-40ff-8e49-23f5674df5ee.png)


AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers. You can trigger Lambda from over 200 AWS services and software as a service (SaaS) applications, and only pay for what you use.
What if you wanted to automate some of these processes in Travis? Well in this short read I'll let you know how in this short and sweet blog.

<!-- more --> 

## Permissions

First, let's create an AWS Role that has the following permissions/conditionals:

```json
{
    "Version": "2021-12-03",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:*"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

Remember when dealing with servies, each service should have a basic handler with the following code, at least in my project a (TypeScript) project:

```typescript
'use strict';
module.exports.hello = async event => {
  return {
    statusCode: 200,
    body: JSON.stringify(
      {
        message: 'Go Serverless v1.0! Your function executed successfully!',
        input: event,
      },
      null,
      2
    ),
  };
// Use this code if you don't use the http event with the LAMBDA-PROXY integration
  // return { message: 'Go Serverless v1.0! Your function executed successfully!', event };
};
```

## Your Travis config 

You now then have most of your permissions locked down, you'll need some of them but in this case it's a bit more secure. Grab your `ACCESS_KEY` for this role in AWS. Now let's focus on the `.travis.yml` file:

In this example, I have a TypeScript project, you'll notice I'm using anchoring in the YAML syntax, which isn't used very often, but it's the superior way to setup your YAML file, I talk about anchoring in another blog post [here](https://blog.travis-ci.com/2021-09-10-flexible). 

```yaml
language: node_js
node_js:
  - "10"
deploy_service_job: &DEPLOY_SERVICE_JOB
  cache:
    directories:
      - node_modules
      - ${SERVICE_PATH}/node_modules
install:
    - npm install -g serverless
    - travis_retry npm install
    - cd ${SERVICE_PATH}
    - travis_retry npm install
    - cd -
   
script:
    - cd ${SERVICE_PATH}
    - serverless deploy -s ${STAGE_NAME}
    - cd -
environments:
  - &PRODUCTION_ENV
    - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID_PRODUCTION}
    - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY_PRODUCTION}
- &UAT_ENV
    - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID_UAT}
    - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY_UAT}
- &DEVELOPMENT_ENV
    - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID_DEVELOPMENT}
    - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY_DEVELOPMENT}
jobs:
  include:
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Users API"
      if: type = push AND branch = develop
      env:
        - SERVICE_PATH="users-api"
        - STAGE_NAME=dev
        - *DEVELOPMENT_ENV
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Test Experiences API"
      if: type = push AND branch = develop
      env:
        - SERVICE_PATH="todo-api"
        - STAGE_NAME=dev
        - *DEVELOPMENT_ENV
# Push @ the UAT branch that deploys to the 'uat' stage
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Test Users API"
      if: type = push AND branch = uat
      env:
        - SERVICE_PATH="montana-api"
        - STAGE_NAME=uat
        - *UAT_ENV
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Test Experiences API"  # I used Regex here
      if: "((branch IN (master, develop) && type = push) OR branch =~ /.*env.*/ OR commit_message
    =~ /\\[recreate env\\]/) AND commit_message !~ /\\[delete env\\]/ AND type !=
    cron AND commit_message !~ /\\[execute .*. test\\]/ AND commit_message !~ /\\[start
    recreate scheduler\\]/"
      env:
        - SERVICE_PATH="todo-api"
        - STAGE_NAME=uat
        - *UAT_ENV
# To the master branch deploys to the 'prod' stage
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Test Users API"
      if: type = push AND branch = master
      env:
        - SERVICE_PATH="users-api"
        - STAGE_NAME=prod
        - *PRODUCTION_ENV
    - <<: *DEPLOY_SERVICE_JOB
      name: "Deploy Test Experiences API"
      if: type = push AND branch = master
      env:
        - SERVICE_PATH="todo-api"
        - STAGE_NAME=prod
        - *PRODUCTION_ENV
```

## Encrypting the AWS Key ID

Let's encrypt our AWS Key ID to the root file. We'll also want to add our `secret` key, but we want to encrypt it. So we'll use the Travis CLI:

```bash
travis encrypt AWS_SECRET_ACCESS_KEY=YOUR_SECRET_ACCESS_KEY --add env.global
```

So we are all set on the Travis end, now we want to build and upload:

```aws
.MONTANAPHONY: build clean upload
build:
	npm install --only=production
	zip -r code.zip . -x *.git*

clean:
	if [ -a code.zip ]; then rm code.zip; fi

upload: build
	./montana.sh
```
Here is the source of the file entitled `montana.sh`:

```bash
#!/bin/bash
aws lambda update-function-code \
--zip-file=fileb://code.zip \
--region=us-east-1 \
--function-name=NAME_OF_YOUR_LAMBDA
echo "Deployed"
```
Now everytime you push to `master` it should update your Lambda to the newest release. 

## Conclusion

There you have it, it was short and sweet, the `.travis.yml` file was a bit complex, but it's a good example for the concept at hand. Remember if you have any questions, any questions at all, please email me at [montana@travis-ci.org](montana@travis-ci.org).

Happy building! 

