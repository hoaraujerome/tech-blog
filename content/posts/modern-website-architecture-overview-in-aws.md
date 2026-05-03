+++
date = '2021-12-09T15:00:19-04:00'
title = 'Modern Website Architecture Overview in AWS'
+++

I have recently launched a new website [snapvocab](https://github.com/hoaraujerome/snapvocab) on the AWS cloud. This hands-on experience allowed me to practice what I learned in the AWS Cloud Developer Certification. After long hours working on it - more than I expected, I can tell you it was worth it. Nothing can ever replace having our hands dirty!

From a functional point of view, it is simply a CRUD application that allows a user to manage a list of words with a paid plan. On the technical side, my goal was to leverage AWS services to go live as soon as possible and at a lower cost.

![Snapvocab Application Architecture Serverles](images/modern_web_architecture_aws.png)

Main design choices:
* Serverless because who wants to manage servers? ;-)
* Stripe, as a payment platform, because of its "no-code" modules that allow to start quickly and accept payments manually.
* Direct access to the database from the browser for simplicity.
* Amplify UI to handle authentication flows seamlessly with Cognito.
* CDK for provisioning the cloud resources with a programming language.
