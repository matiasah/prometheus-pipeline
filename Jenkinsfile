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
        string(description: "Istio Ingress Gateway", name: "ISTIO_INGRESS_GATEWAY", defaultValue: env.ISTIO_INGRESS_GATEWAY ? env.ISTIO_INGRESS_GATEWAY : '')
        string(description: "Istio Host", name: "ISTIO_HOST", defaultValue: env.ISTIO_HOST ? env.ISTIO_HOST : '')
        booleanParam(description: "Debug", name: "DEBUG", defaultValue: env.DEBUG ? env.DEBUG : "false")
        booleanParam(description: "Restart", name: "RESTART", defaultValue: env.RESTART ? env.RESTART : "false")
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
                  - command:
                    - "cat"
                    image: "k8s.gcr.io/kustomize/kustomize:v5.0.1"
                    imagePullPolicy: "IfNotPresent"
                    name: "kustomize"
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
                        sh "kubectl apply -f prometheus-namespace.yaml"
                        sh "kubectl apply -f prometheus-adapter-namespace.yaml"

                    }

                }

            }

        }

        stage ("Prometheus: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Prometheus options
                        PROMETHEUS_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            PROMETHEUS_OPTIONS += "--debug "

                        }

                        // Template
                        sh "helm template prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml ${PROMETHEUS_OPTIONS.trim()} --namespace prometheus --include-crds > prometheus-base.yaml"
        
                    }

                }

                container ("kustomize") {
  
                    script {

                        // Download envsubst
                        httpRequest url: "https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-Linux-x86_64", outputFile: "envsubst"

                        // Add execution permission to envsubst
                        sh "chmod +x envsubst"
                        sh "ls -l -a"
    
                        sh '''
                        for file in ./custom-resource/*; do
                            ./envsubst < "${file}" > out.txt && mv out.txt "${file}";
                        done
                        '''
  
                        // Kustomize
                        sh "kustomize build > prometheus-template.yaml"

                        // Print Yaml
                        sh "cat prometheus-template.yaml"
  
                    }
  
                }

            }

        }

        stage ("Prometheus: Apply") {

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

        stage ("Prometheus Adapter: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Prometheus options
                        PROMETHEUS_ADAPTER_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            PROMETHEUS_ADAPTER_OPTIONS += "--debug "

                        }

                        // Template
                        sh "helm template prometheus-adapter prometheus-community/prometheus-adapter -f prometheus-adapter-values.yaml ${PROMETHEUS_ADAPTER_OPTIONS.trim()} --namespace prometheus-adapter --api-versions apiregistration.k8s.io/v1 > prometheus-adapter-template.yaml"

                        // Print Yaml
                        sh "cat prometheus-adapter-template.yaml"
        
                    }

                }

            }

        }

        stage ("Prometheus Adapter: Apply") {

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
                            sh "kubectl apply -f prometheus-adapter-template.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f prometheus-adapter-template.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm prometheus-adapter-template.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan") && env.RESTART.equals("true")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n prometheus"
                        sh "kubectl rollout restart daemonset -n prometheus"

                        sh "kubectl rollout restart deployment -n prometheus-adapter"
                        sh "kubectl rollout restart daemonset -n prometheus-adapter"

                    }

                }

            }

        }

    }
    
}
