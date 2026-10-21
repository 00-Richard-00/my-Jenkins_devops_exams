pipeline { agent any

environment {
    // Declaration of environment variables
    DOCKER_ID = "0jr0"
    DOCKER_IMAGE_MOVIE = "exam-jenkins-movie"
    DOCKER_IMAGE_CAST = "exam-jenkins-cast"
    DOCKER_TAG = "v.${BUILD_ID}.0"

    HELM_RELEASE = "my-app"
    HELM_CHART = "./helm/my-app"
}

stages {

stage('Debug branche') {
    steps {
        echo "BRANCH_NAME = '${env.BRANCH_NAME}'"
        echo "GIT_BRANCH  = '${env.GIT_BRANCH}'"
        echo "GIT_LOCAL_BRANCH = '${env.GIT_LOCAL_BRANCH}'"
    }
}

    stage('Build des images Movie et Cast') {
        steps {
            sh '''
                set -e
                docker build -t "$DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG" ./movie-service
                docker build -t "$DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG" ./cast-service
            '''
        }
    }

    stage('Docker Push') {
        environment {
            DOCKER_PASS = credentials('DOCKER_HUB_PASS')
        }
        steps {
            sh '''
                set -e
                echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
                docker push "$DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG"
                docker push "$DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG"
            '''
        }
    }

    stage('Deploy DEV') {
        steps {
            script {
                deployToNamespace('exam-jenkins-dev')
            }
        }
    }

    stage('Deploy STAGING') {
        steps {
            script {
                deployToNamespace('exam-jenkins-staging')
            }
        }
    }

    stage('Deploy QA') {
        steps {
            script {
                deployToNamespace('exam-jenkins-qa')
            }
        }
    }

    stage('Approbation PROD') {
        when {
            expression {
                env.BRANCH_NAME == 'master'
            }
        }
        steps {
            timeout(time: 30, unit: 'MINUTES') {
                input(
                    message: "Autoriser le déploiement ${DOCKER_TAG} en PROD ?",
                    ok: 'Valider le déploiement',
                    submitter: 'jenkins-admin'
                )
            }
        }
    }

    stage('Deploy PROD') {
        when {
            expression {
                env.BRANCH_NAME == 'master'
            }
        }
        steps {
            script {
                deployToNamespace('exam-jenkins-prod')
            }
        }
    }
}


}

def deployToNamespace(String namespace) { sh """ set -e

    helm upgrade --install "${HELM_RELEASE}" "${HELM_CHART}" \
      --namespace "${namespace}" \
      --create-namespace \
      --set movie.image.tag="${DOCKER_TAG}" \
      --set cast.image.tag="${DOCKER_TAG}" \
      --wait \
      --timeout 5m

    kubectl rollout status deployment/movie-service \
      -n "${namespace}" --timeout=180s

    kubectl rollout status deployment/cast-service \
      -n "${namespace}" --timeout=180s

    kubectl rollout status deployment/nginx \
      -n "${namespace}" --timeout=180s

    kubectl rollout status deployment/movie-db \
      -n "${namespace}" --timeout=180s

    kubectl rollout status deployment/cast-db \
      -n "${namespace}" --timeout=180s

    kubectl get deployments,pods,pvc -n "${namespace}"
"""


}