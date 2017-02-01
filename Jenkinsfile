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
		DOCKER_CONTAINER_ID=$(docker run -d deepdiver/docker-oracle-xe-11g)
		DATABASEHOST=$(docker inspect --format="{{.NetworkSettings.IPAddress}}" "$DOCKER_CONTAINER_ID")

		php --version

		composer install

		cat > tests/travis/oracle.travis.xml <<DELIM
		<?xml version="1.0" encoding="utf-8"?>
		<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
			 xsi:noNamespaceSchemaLocation="http://schema.phpunit.de/4.8/phpunit.xsd"
			 backupGlobals="false"
			 colors="true"
			 bootstrap="../../vendor/autoload.php"
		>
		    <php>
			<ini name="error_reporting" value="-1" />

			<var name="db_type" value="oci8"/>
			<var name="db_host" value="$DATABASEHOST" />
			<var name="db_username" value="doctrine" />
			<var name="db_password" value="oracle" />
			<var name="db_name" value="XE" />
			<var name="db_port" value="1521"/>

			<var name="db_event_subscribers" value="Doctrine\DBAL\Event\Listeners\OracleSessionInit" />

			<var name="tmpdb_type" value="oci8"/>
			<var name="tmpdb_host" value="$DATABASEHOST" />
			<var name="tmpdb_username" value="system" />
			<var name="tmpdb_password" value="oracle" />
			<var name="tmpdb_port" value="1521"/>
		    </php>

		    <testsuites>
			<testsuite name="Doctrine DBAL Test Suite">
			    <directory>../Doctrine/Tests/DBAL</directory>
			</testsuite>
		    </testsuites>

		    <groups>
			<exclude>
			    <group>performance</group>
			    <group>locking_functional</group>
			</exclude>
		    </groups>
		</phpunit>

		DELIM

		sleep 5m

		./vendor/bin/phpunit --configuration tests/travis/oracle.travis.xml --log-junit phpunit-results.xml

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


