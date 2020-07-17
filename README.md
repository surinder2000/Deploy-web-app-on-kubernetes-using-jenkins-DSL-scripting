# Deploy web app on top of kubernetes using Jenkins DSL scripting
Deploy web app on top of kubernetes using jenkins and create jobs in jenkins using jenkins DSL (Domain Specific Language) scripting 

## Pre-requisites
* Must have minikube installed
* Must have kubectl configured

### Create Jenkins container image with kubectl configured
Following set of commands used for creating Jenkins container image with kubectl configured

    FROM centos
    RUN yum install sudo -y
    RUN yum install net-tools -y
    RUN yum install wget -y
    RUN yum install git -y
    RUN yum install curl -y
    RUN yum install openssh-clients -y

    RUN wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    RUN rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
    RUN yum install java-11-openjdk.x86_64 -y
    RUN yum install jenkins -y
    RUN echo -e "jenkins ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers 
    CMD /etc/rc.d/init.d/jenkins start 

    RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    RUN chmod +x ./kubectl
    RUN cp ./kubectl /usr/bin/
    RUN mkdir /root/.kube
    COPY client.key /root/.kube
    COPY client.crt /root/.kube
    COPY ca.crt /root/.kube
    COPY config /root/.kube
    EXPOSE 8080
    CMD ["java","-jar","/usr/lib/jenkins/jenkins.war"]
    
Run the following command to build the image

    docker build -t surinder2000/jenkins-kube:v1.0
    
Run the following command to push the image into Docker hub

    docker login
    docker push surinder2000/jenkins-kube:v1.0
    
    
### Create Jenkins deployment with persistent storage and expose it
Follwing code is used for creating Jenkins deployment with persistent storage and for exposing it

    apiVersion: v1
    kind: Service
    metadata:
      name: jenkins-deploy 
      labels:
        app: jenkins
    spec:
      ports:
        - port: 8080
      selector:
        app: jenkins 
      type: NodePort

    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: jenkins-pv-claim
      labels:
        app: jenkins 
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
    ---
    apiVersion: apps/v1 
    kind: Deployment
    metadata:
      name: jenkins-deploy 
      labels:
        app: jenkins 
    spec:
      selector:
        matchLabels:
          app: jenkins 
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: jenkins 
        spec:
          containers:
          - image: surinder2000/jenkins-kube:v1.0
            name: jenkins 
            ports:
            - containerPort: 8080
              name: jenkins 
            volumeMounts:
            - name: jenkins-storage
              mountPath: /root/.jenkins 
          volumes:
          - name: jenkins-storage
            persistentVolumeClaim:
              claimName: jenkins-pv-claim

Run the following command to deploy Jenkins on top of Kubernetes

    kubectl create -f jenkins-deploy.yml 
    
Here, jenkins-deploy.yml is the deployment file of Jenkins

To get the console use **minikubeIP:nodePort** in the browser. To get the node port run the following command

    kubectl get svc/jenkins-deploy
    
Configure the jenkins and install the required plugins. Must install DSL job and Build  pipeline plugin

To install plugins go to **Manage Jenkins -> Manage Plugins** click on available, search for required plugins and install them

### Create Jenkins jobs using DSL scripting
#### Job1: Pull the code from Github when some developer push code to Github repository
Following is the code for creating Job1

    job("Pull data from Github"){
      description("Pull the data from github repo automatically when some developers push code to github")
      scm{
        github("surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting","master")
      }
      triggers {
        scm("* * * * *")
      }
      steps{
        shell('''if ls /root | grep webdata
    then
    sudo cp -rf * /root/webdata
    else
    mkdir /root/webdata
    sudo cp -rf * /root/webdata
    fi  
    ''')
      }
    }
    
#### Job2: By looking at the code file name launch the deployment of respective webserver
Following is the code for creating Job2

    job("Launch deployment"){
      description("By looking at the code it will launch the deployment of respective webserver and the deployment will launch webserver, create PVC and expose the deployment")

      triggers {
        upstream("Pull data from Github", "SUCCESS")
      }
      steps{
        shell('''data=$(sudo ls /root/webdata)
    if sudo ls /root/webdata/ | grep html
    then
    if sudo kubectl get deploy/html-webserver
    then
    echo "Already running"
    POD=$(sudo kubectl get pod -l server=apache-httpd -o jsonpath="{.items[0].metadata.name}")
    for file in $data
    do
    sudo kubectl cp /root/webdata/$file $POD:/usr/local/apache2/htdocs/
    done
    else
    sudo kubectl create -f /root/webdata/htmlweb.yml
    POD=$(sudo kubectl get pod -l server=apache-httpd -o jsonpath="{.items[0].metadata.name}")
    sleep 30
    for file in $data
    do
    sudo kubectl cp /root/webdata/$file $POD:/usr/local/apache2/htdocs/
    done
    fi
    elif sudo ls /root/webdata/ | grep php
    then
    if sudo kubectl get deploy/php-webserver
    then
    echo "Already running"
    POD=$(sudo kubectl get pod -l server=apache-httpd-php -o jsonpath="{.items[0].metadata.name}")
    for file in $data
    do
    sudo kubectl cp /root/webdata/$file $POD:/var/www/html/
    done
    else
    sudo kubectl create -f /root/webdata/phpweb.yml
    POD=$(sudo kubectl get pod -l server=apache-httpd -o jsonpath="{.items[0].metadata.name}")
    sleep 30
    for file in $data
    do
    sudo kubectl cp /root/webdata/$file $POD:/var/www/html/
    done
    fi
    else
    echo "No server found"
    exit 1
    fi
    ''')
      }
    }

#### Job3: Check whether the site is working or not if not working send email notification
Following is the code for creating Job3

    job("Check status of site"){
      description("Check whether the web app is working or not. If it is not working sent email to developer")
      triggers {
        upstream("Launch deployment", "SUCCESS")
      }
      steps{
        shell('''if sudo ls /root/webdata | grep html
    then
    status=$(sudo curl -o /dev/null -s -w "%{http_code}" 192.168.99.111:31001)
    elif sudo ls /root/webdata | grep php
    then
    status=$(sudo curl -o /dev/null -s -w "%{http_code}" 192.168.99.111:31002)
    fi
    if [[ status -ne 200 ]]
    then 
    if sudo ls /root/webdata/ | grep html
    then
    sudo kubectl delete -f /root/webdata/htmlweb.yml
    exit 1
    elif sudo ls /root/webdata/ | grep php
    then
    sudo kubectl delete -f /root/webdata/phpweb.yml
    exit 1
    fi
    else
    exit 0
    fi
    ''')
      }
      publishers {
        extendedEmail {
          recipientList("surinderkumarmanhas901@gmail.com")
          defaultSubject("Status of site")
          defaultContent("Site is not working fine. Please check the code")
          contentType("text/plain")
          triggers {
            failure {
              sendTo {
                recipientList()
              }
            }
          }
        }
      }
    }



#### Job4: Send email notification to developer if site is working fine
Following is the code for creating Job4

    job("Email notification"){
      description("Send email notification to developer if site is working fine")

      triggers {
        upstream("Check status of site", "SUCCESS")
      }
      publishers {
        extendedEmail {
          recipientList("surinderkumarmanhas901@gmail.com")
          defaultSubject("Status of site")
          defaultContent("Site is working fine")
          contentType("text/plain")
          triggers {
            always {
              sendTo {
                recipientList()
              }
            }
          }
        }
      }
    }
    
#### Create a Build pipeline view of the jobs
Following is the code for creating pipeline view of the jobs

    buildPipelineView("Pipeline view") {
      filterBuildQueue(true)
      filterExecutors(false)
      title("My pipeline")
      displayedBuilds(1)
      selectedJob("Pull data from Github")
      alwaysAllowManualTrigger(true)
      showPipelineParameters(true)
      refreshFrequency(10)
    }
    
    
### Create a seed job in jenkins that pull the DSL script from Github and process the DSL script for creating the jobs
* In Source Control Management section put the Github repository url and branch name

![Git configuration](https://github.com/surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting/blob/master/Screenshots/SeedJob1.png)

* In Build trigger section select Poll SCM for checking the github repository every minute

![Build trigger](https://github.com/surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting/blob/master/Screenshots/SeedJob2.png)

* In the Build section from Add build step select Process Job DSLs, check Look on Filesystem and put the name of the file in DSL Scripts box

![Process DSL script](https://github.com/surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting/blob/master/Screenshots/SeedJob3.png)

Now as soon as the DSL script is pushed into the Github repositroy by some developer, the seed job pull the code from Github repository and creates other Jenkins jobs

After successful Build of seed Job following jobs created

![Jobs created](https://github.com/surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting/blob/master/Screenshots/JobsCreated.png)

The created jobs deploys web app on top of Kubernetes as soon as the developer push the code into the Github repository

This is the build pipeline view of the Jobs created by Jenkins DSL scritp

![Pipelineview](https://github.com/surinder2000/Deploy-web-app-on-kubernetes-using-jenkins-DSL-scripting/blob/master/Screenshots/Pipelineview.png)


[Jenkins jobs code link](https://github.com/surinder2000/jenkins-DSL-script)

### Thank you!
