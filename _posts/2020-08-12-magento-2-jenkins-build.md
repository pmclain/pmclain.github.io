---
layout: post
title:  "Building Magento 2 with Jenkins"
date:   2020-08-12 06:00:00 -0500
categories: magento2,devops
---

# Building Magento 2 with Jenkins

I often find myself needing a clean Magento 2 environment. The version required
generally depends on what I'm doing and means I switch relatively often. I'm
lazy and wanted something that just works by pushing a button or two. I had
stood up a Jenkins instance the prior week and thought I'd use that for creating
the build artifacts and eventually deploying. Here I will focus on the build and
archive generation process.

Since I'm often switching between similar versions I want to avoid the build
process when deploying if possible. For example, if I've already built 2.4.0 and
2.3.5 I should not need to rebuild them when switching versions. I found a few
different approaches such as writing the archive to AWS S3 or Artifactory. I
wanted to use my existing infrastructure and decided on installing the
[`copyArtifacts` Jenkins plugin](https://plugins.jenkins.io/copyartifact/){:target="_blank"}.

We'll be performing the build withing a docker container based on one of the
official PHP images `php:7.3-cli`. You can view the Dockerfile on [GitLab](https://github.com/pmclain/m2-jenkins-deploy/blob/master/docker/Dockerfile){:target="_blank"}.
The final version is very basic. We install a few packages for meeting Magento's
minimum requirements along with composer.

An oddity within the Dockerfile is copying a generic store configuration. I'm
performing the build from the Magento GitHub repository for running develop
branches between releases. This means I won't have a complete `config.php` file
available at build time. `COPY files/store-config.php /var/store-config.php`
will fill in the gaps for running `setup:static-content:deploy`.

With the Dockerfile ready we'll start writing the Jenkinsfile used for the
build. Here's the initial pipeline for preparing the workspace for build. The
stage is configured for running inside the PHP Docker container.

* Build parameters for inputting the tag or branch and select letting the
  pipeline know if the input is a tag or branch.
* Add Magento composer auth from a Jenkins stored credential.
* Checkout the Magento source from GitHub using the provided reference.

{% highlight groovy %}
pipeline {
    agent {
        node {
            label 'master'
        }
    }

    parameters {
        choice(
            name: 'reference_type',
            choices: [
                'tag',
                'branch'
            ]
        )
        string(
            name: 'reference',
            defaultValue: '2.4.0'
        )
    }

    stages {
        stage('Build') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir 'docker/'
                    label 'master'
                    args '-v $HOME/jenkins_home/.composer:$HOME/.composer'
                }
            }

            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "magento-repo", // user/pass credential
                            usernameVariable: "username",
                            passwordVariable: "password"
                        )
                    ]) {
                        sh """
                            composer global config -a http-basic.repo.magento.com "$username" "$password"
                        """
                    }
                }

                script {
                    if ( "${env.reference_type}" == 'branch' ) {
                        env.build_from = "refs/heads/${env.reference}"
                    }

                    else if ( "${env.reference_type}" == 'tag' ) {
                        env.build_from = "refs/tags/${env.reference}"
                    }
                }

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${env.build_from}"]],
                    userRemoteConfigs: [[name: 'source', url: "https://github.com/magento/magento2.git"]]
                ])
            }
        }
    }
}
{% endhighlight %}

We can now add our build commands into the stage. The build steps are basic:

* Clear `generated/code`, `generated/metadata` and `pub/static`
* `composer install` with the `--optimize-autoloader --no-interaction` options
* Create a `config.php` by enabling all modules. You would not do this with a
  standard Magento project. Your `config.php` __should__ be commited to source
  control.
* A hack merging the new `config.php` with a boilerplate store config. Again
  this is only needed because it isn't a proper project.
* Run `setup:di:compile` and `setup:static-content:deploy`
* Clear `.git`. I don't usually do this, but it adds 250MB of bloat to the build
  artifact. In this case it isn't worth the overhead.
* Create an archive of the whole thing.

Here's the end result

{% highlight groovy %}
    sh 'rm -rf generated/code generated/metadata pub/static/frontend pub/static/adminhtml'
    sh 'composer install --optimize-autoloader --no-interaction'
    sh 'php bin/magento module:enable --all'
    sh 'php -r \'$config = include("app/etc/config.php"); $stores = include("/var/store-config.php"); file_put_contents("app/etc/config.php", "<?php\\nreturn\\n" . var_export(array_merge($config, $stores), true) . ";");\''
    sh 'php bin/magento setup:di:compile'
    sh 'php bin/magento setup:static-content:deploy -f'
    sh 'rm -rf .git/'
    sh 'mkdir build'
    sh 'tar --exclude=build/output.tar.gz -zcf build/output.tar.gz .'
{% endhighlight %}

Finally, I need Jenkins to archive the artifact and define which jobs can access
via `copyArtifacts`. Right after the last build step I add:

{% highlight groovy %}
archiveArtifacts artifacts: 'build/output.tar.gz', followSymlinks: false, onlyIfSuccessful: true
{% endhighlight %}

letting Jenkins know the location of the build artifact. Then adding the
following to the pipeline configuration before the parameters:

{% highlight groovy %}
options {
    copyArtifactPermission('m2-demo-store/DeployDemoStore')
}
{% endhighlight %}

That a job named `m2-demo-store/DeployDemoStore` access to the build artifact.
I'll work on the deploy job another day. You can see find the complete
[Jenkinsfile on GitHub](https://github.com/pmclain/m2-jenkins-deploy/blob/a4fb8cac2ebe1ee9c26ecdad1147b0160938934e/jenkins/MagentoDemoBuild){:target="_blank"}
