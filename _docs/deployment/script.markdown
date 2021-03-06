---
layout:         doc
title:          "Script deployment - Documentation"
category:       "deployment"
logo:           script
order:          2
excerpt:        "A generic deployment type where you can specify your own deployment script."
---

This deployment type allows you to deploy to your servers by using your own deployment script. You can use the package
generated by continuousphp or you can use the repository's source code directly.

## Configure the deployment
To configure the deployment, you only need to specify your script(s) in the last step of the pipeline configuration.
You can either a script that is part of your repository's code or execute shell commands directly.

![Script Deployment configuration](/assets/doc/deployment/script/configuration.png)

## Package and source code
Before executing your scripts, the application package will automatically be downloaded and extracted. continuousphp will then
change automatically the current working directory to the root of your application. This way, you can execute your script
commands without changing the path beforehand.

Alternatively, you can use the environment variable `PACKAGE_PATH` to get the path to the deployment package.

