#!groovy
/*
 * This Jenkinsfile is intended to run on https://ci.owncloud.org and may fail anywhere else.
 * It makes assumptions about plugins being installed, labels mapping to nodes that can build what is needed, etc.
 */

timestampedNode('SLAVE') {
    stage 'Checkout'
        checkout scm
        sh '''composer install'''

    stage 'PHP 5.6'
        executeAndReport('phpunit-results.xml') {
            sh '''
		phpenv local 5.6
		php --version

		DOCKER_CONTAINER_ID=$(docker run -d deepdiver/docker-oracle-xe-11g)
		DATABASEHOST=$(docker inspect --format="{{.NetworkSettings.IPAddress}}" "$DOCKER_CONTAINER_ID")

		sleep 5m

		./test/jenkins/run.sh $DATABASEHOST

		./vendor/bin/phpunit --configuration tests/jenkins/oracle.travis.xml --log-junit phpunit-results.xml

		docker stop $DOCKER_CONTAINER_ID || true
		docker kill $DOCKER_CONTAINER_ID || true
            '''
        }
}

void executeAndReport(String testResultLocation, def body) {
    def failed = false
    // We're wrapping this in a timeout - if it takes longer, kill it.
    try {
        timeout(time: 120, unit: 'MINUTES') {
            body.call()
        }
    } catch (Exception e) {
        failed = true
        echo "Test execution failed: ${e}"
    } finally {
        step([$class: 'JUnitResultArchiver', testResults: testResultLocation])
    }
}

// Runs the given body within a Timestamper wrapper on the given label.
def timestampedNode(String label, Closure body) {
    node(label) {
        wrap([$class: 'TimestamperBuildWrapper']) {
            body.call()
        }
    }
}


