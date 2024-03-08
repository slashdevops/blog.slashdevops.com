+++
title = 'Secure and Easy AWS Connection Assuming a Role With Jenkins Shared Library'
description = 'How to connect to AWS Account assuming a role with Jenkins shared library'
date = 2022-12-03T20:39:29+01:00
draft = true
+++
## The Problem

Surely and like me, you are trying to be more secure when connecting [Jenkins](https://www.jenkins.io/) with your [AWS Accounts](https://aws.amazon.com/account/) assuming a role. If you are asking What is that? , please read this: [https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)

Of course, there are many different options to use, but the problem always surrounds us, if you use a plugin then the maintainability and security when talking about Jenkins plugins for sure decrease.

I particularly hate Jenkins, from my point of view this is an obsolete tool trying to survive in the modern world, and if you are concerned about security (and maintainability) sure understand my point.

## So, why I’m writing about that?

+ Because unfortunately I still using Jenkins and sweating their maintenance
+ Because as a rule of thumb I try to avoid plugins that don’t have any release in the time windows of 6-12 months
+ Helps others to avoid loose time and security when needs the same that me, AWS cross-account connections using Jenkins assuming a role
+ Because at least if I have an Issue this is my code and I can fix it

## What’s this?

+ This is a guide and code for somebody using Jenkins shared library
+ This is a minimal blog entry to help someone that understands Jenkins and Groovy
+ This could help you if you are using Jenkins + Jenkins shared library + AWS cross-account and cross-region roles

## what it is not?

+ A tutorial
+ A very well-explained and step-by-step guide
+ Something you surely need to use
+ An AWS cross-account tutorial or explanation guide

## The Solution

This is how looks a segment of the code on my production [Jenkins declarative pipeline](https://www.jenkins.io/doc/book/pipeline/syntax/).

Look at __withAwsEnVars__ (lines: 4, 11) pipeline tags

```groovy {linenos=table,hl_lines=[4,11]}
...
      stage('setup repositories') {
        steps {
          withAwsEnVars(roleName: cicd.getRole(), roleAccount: codeArtifact.getOwner('npm-private')) {
            script {
              log.info('setting repositories \'npm-private\' credentials for dependencies')
              codeArtifact.setupNpmrc('npm-private', '@my-company-namespace', params.timeoutTime*60)
              codeArtifact.setupNpmrc('npm-private', '@my-company-other-namespace', params.timeoutTime*60)
            }
          }
          withAwsEnVars(roleName: cicd.getRole(), roleAccount: codeArtifact.getOwner('npm-public')) {
            script {
              log.info('setting repositories \'npm-public\' credentials for dependencies')
              codeArtifact.setupNpmrc('npm-public', '', params.timeoutTime*60)
            }
          }
        }
      }
...
```

__withAwsEnVars__ is a [Groovy function](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#directory-structure) used in my [Jenkins Shared Library](https://github.com/christiangda/jenkins-shared-library) and this is the [withAwsEnVars.groovy](https://github.com/christiangda/jenkins-shared-library/vars/withAwsEnVars.groovy) file content:

```groovy {linenos=table}
#!/usr/bin/env groovy

/*
paramters:
  roleArn (required)
  roleAccount (required)
  sessionName (optional)
  sessionDuration (optional)

Examples:
  stage('test aws credential') {
    steps {
      withAwsEnVars(roleName:'cicd-execution-role', roleAccount: '12345678910') {
        sh "echo TOKEN: ${AWS_SESSION_TOKEN}"
        sh "echo KEY: ${AWS_SECRET_ACCESS_KEY}"
        sh "echo ID: ${AWS_ACCESS_KEY_ID}"
        sh 'aws s3 ls'
      }
      sh "exit 1"
    }
  }
*/
def call(Map params, Closure body) {

  if (!params.roleName) {
    error """
      parameter 'roleName' is required.
      ---
      Example:  withAwsEnVars(roleName:'cicd-execution-role', roleAccount: '12345678910') {...}
    """
  }

  if (!params.roleAccount) {
    error """
      parameter 'roleAccount' is required.
      ---
      Example:  withAwsEnVars(roleName:'cicd-execution-role', roleAccount: '12345678910') {...}
    """
  }

  // get optional parameters if not set default
  String sessionName = params.get('sessionName', 'jenkins')
  Integer duration = params.get('sessionDuration', 900)

  cred = awsCredentials.getFromAssumeRole(params.roleName, params.roleAccount, sessionName, duration)

  AWS_ACCESS_KEY_ID = cred.AccessKeyId
  AWS_SECRET_ACCESS_KEY = cred.SecretAccessKey
  AWS_SESSION_TOKEN = cred.SessionToken

  wrap([
      $class: 'MaskPasswordsBuildWrapper',
      varPasswordPairs: [
        [password: AWS_ACCESS_KEY_ID],
        [password: AWS_SECRET_ACCESS_KEY],
        [password: AWS_SESSION_TOKEN]
      ]
  ]) {
    withEnv([
      "AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}",
      "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}",
      "AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
    ]) {
      body()
    }
  }
}
```

and this is my [awsCredentials.groovy](https://github.com/christiangda/jenkins-shared-library/vars/awsCredentials.groovy) file content:

```groovy {linenos=table}
#!/usr/bin/env groovy

/*
Parameters:
  roleName (required)
  roleAccount (required)

Examples:
  cred = awsCredentials.getFromAssumeRole(...)
  accessKey = awsCredentials.getFromAssumeRole(...).AccessKeyId
*/
String getFromAssumeRole(String roleName, String roleAccount, String sessionName='jenkins', Integer duration=900){

  String roleArn = 'arn:aws:iam::' + roleAccount +':role/'+ roleName

  List<String> options = []

  options += "--role-arn ${roleArn}"
  options += "--role-session-name ${sessionName}"
  options += "--duration-seconds ${duration}"
  options += "--query 'Credentials'"

  optionsString = options.join(" ")

  // this is used to mask any critical information
  wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[password: roleArn], [password: sessionName]]]) {
    String strCreds = sh(
      returnStdout: true,
      script: """
        aws sts assume-role ${optionsString}
        """).trim()

    return readJSON(text: strCreds)
  }
}
```

from the code above, definitions are located in the [Jenkins Shared Library](https://github.com/christiangda/jenkins-shared-library)
+ [codeArtifact.setupNpmrc(...)](https://github.com/christiangda/jenkins-shared-library/blob/bd79251051606d05d53bcefe3c6c376c1971620b/vars/codeArtifact.groovy#L92) -> [codeArtifact.groovy](https://github.com/christiangda/jenkins-shared-library/blob/main/vars/codeArtifact.groovy)
+ [codeArtifact.getOwner(...)](https://github.com/christiangda/jenkins-shared-library/blob/bd79251051606d05d53bcefe3c6c376c1971620b/vars/codeArtifact.groovy#L18) -> [codeArtifact.groovy](https://github.com/christiangda/jenkins-shared-library/blob/main/vars/codeArtifact.groovy)
+ [cicd.getRole()](https://github.com/christiangda/jenkins-shared-library/blob/main/vars/cicd.groovy#L6) -> [cicd.groovy](https://github.com/christiangda/jenkins-shared-library/blob/main/vars/cicd.groovy)

### The minimal requirements on your Jenkins controller and agents

+ [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
+ [Jenkins Plugin – Pipeline Utility Steps](https://plugins.jenkins.io/pipeline-utility-steps/) –> [How to use?](https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/#pipeline-utility-steps)
+ [Jenkins Plugin – Pipeline: Basic Steps](https://plugins.jenkins.io/workflow-basic-steps/) –>[How to use?](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#pipeline-basic-steps)
+ [Jenkins Plugin – Mask Passwords](https://plugins.jenkins.io/mask-passwords/#documentation) –> [How to use?](https://www.jenkins.io/doc/pipeline/steps/mask-passwords/#mask-passwords-plugin)

## So, What is the Magic? Why do I say this is secure?

Maybe after looking at the following pipeline code, you will see how easy is to use this, and this is __secure__ because if you execute the following code in your pipeline:

```groovy {linenos=table}
  ...
   stage('test aws credential') {
     steps {
       withAwsEnVars(roleName:'cicd-execution-role', roleAccount: '12345678910') {
         sh "echo TOKEN: ${AWS_SESSION_TOKEN}"
         sh "echo KEY: ${AWS_SECRET_ACCESS_KEY}"
         sh "echo ID: ${AWS_ACCESS_KEY_ID}"
         sh 'aws s3 ls'
       }
       sh "exit 1"
     }
   }
...
```

you will see __masked__ the __TOKEN__, __KEY__ and __ID__. Instead of seeing the real value, you will see __***********__ characters.

## Tools and Concepts

There are various tools and concepts I used here, the first that allows me to do that so __easily__ was [Groovy Closures](https://groovy-lang.org/closures.html) and it is explained how to use on [Jenkins Shared Library](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#defining-custom-steps) –> [Defining custom steps](https://www.jenkins.io/doc/book/pipeline/shared-libraries/#defining-custom-steps).

Then we have the tool [withEnv](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#withenv-set-environment-variables) provided by the [Jenkins Plugin – Pipeline: Basic Steps](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#pipeline-basic-steps) and in combination with the use of the [Groovy Closures](https://groovy-lang.org/closures.html) allowed me to export the [AWS Environment Variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) coming from [awsCredentials.getFromAssumeRole(…)](https://github.com/christiangda/jenkins-shared-library/blob/9fd0250d914b681696dfd5a097dd935da56589fc/vars/awsCredentials.groovy#L12) groovy function into a container script part.

But, make sure that anyone who will use Jenkins Controller and Pipeline doesn’t have access to the values of the [AWS Environment Variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) is the job of [wrap](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#wrap-general-build-wrapper) provided by the [Jenkins Plugin – Pipeline: Basic Steps](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/#pipeline-basic-steps) + [maskPasswords](https://www.jenkins.io/doc/pipeline/steps/mask-passwords/#mask-passwords-plugin) provided by [Jenkins Plugin – Mask Passwords](https://plugins.jenkins.io/mask-passwords/#documentation).

## Others Links

+ <https://www.jenkins.io/doc/book/installing/>
+ <https://www.jenkins.io/doc/book/using/>
+ <https://www.jenkins.io/doc/book/security/#securing-jenkins>
+ <https://www.jenkins.io/doc/book/pipeline/>

## Closing

Even hating [Jenkins](https://www.jenkins.io/) like me, you can find different ways to do your life easy and secure with him.

If you want to look at my GitHub repositories related to Jenkins, here you have:

+ <https://github.com/christiangda/jenkins-casc-controller>
+ <https://github.com/christiangda/jenkins-shared-library>
