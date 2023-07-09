/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-slave
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-slave-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-slave
  namespace: jenkins
*/

pipeline {

    parameters {
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply", "Destroy"])
        booleanParam(description: "Debug", name: "DEBUG", defaultValue: env.DEBUG ? env.DEBUG : "false")
    }

    agent {
        kubernetes {
            yaml """
                apiVersion: "v1"
                kind: "Pod"
                spec:
                  securityContext:
                    runAsUser: 1001
                    runAsGroup: 1001
                    fsGroup: 1001
                  containers:
                  - command:
                    - "cat"
                    image: "alpine/helm:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "helm"
                    resources: {}
                    tty: true
                    volumeMounts:
                    - mountPath: "/.config"
                      name: "config-volume"
                      readOnly: false
                    - mountPath: "/.cache/helm/"
                      name: "cache-volume"
                      readOnly: false
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
                  volumes:
                  - emptyDir:
                      medium: ""
                    name: "config-volume"
                  - emptyDir:
                      medium: ""
                    name: "cache-volume"
            """
        }
    }

    stages {

        stage ("Helm Repo") {

            steps {

                container ("helm") {

                    script {
    
                        // Install repo
                        sh "helm repo add prometheus-community https://prometheus-community.github.io/helm-charts"
                        sh "helm repo update"
    
                    }

                }

            }

        }

        stage ("Namespace: Apply") {

            when {

                expression {
                    return env.ACTION.equals("Apply")
                }

            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        sh "kubectl create namespace prometheus --dry-run=client -o yaml | kubectl apply -f -"

                    }

                }

            }

        }

        stage ("Template: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Cluster agent options
                        PROMETHEUS_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            PROMETHEUS_OPTIONS += "--debug "

                        }

                        // Template
                        sh "helm template prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml ${PROMETHEUS_OPTIONS.trim()} --namespace prometheus > prometheus-template.yaml"

                        // Print Yaml
                        sh "cat prometheus-template.yaml"
        
                    }

                }

            }

        }

        stage ("Template: Apply") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply
                            sh "kubectl apply -f prometheus-template.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f prometheus-template.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm prometheus-template.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n prometheus"
                        sh "kubectl rollout restart daemonset -n prometheus"

                    }

                }

            }

        }

    }
    
}
