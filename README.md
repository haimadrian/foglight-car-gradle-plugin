# fglam-gradle-plugin
A Foglight Gradle plugin for cartridges team

Developed using Gradle 7.4.2

Plugin is published to https://artifactory.labs.quest.com/ui/repos/tree/General/libs-release-local/com/quest/foglight/fglam

### Usage
#### Add our maven repo, and declare dependency
    buildscript {
        repositories {
            maven {
                url "$artifactoryUrl/libs-release"
                credentials { 
                    username artifactoryUser
                    password artifactoryAPIKey
                }
            }
        }

        dependencies {
            classpath "com.quest.foglight.fglam:fglam-gradle-plugin:1.0"
        }
    }

    apply plugin: 'com.quest.foglight.fglam'

#### Customize extensions
Those are the defaults

    fglam {
        dependenciesOutDir = 'jars'
    }

    fglamVersions {
        log4jVersion = '2.17.1'
        junitVersion = '4.13.2'
        mockitoVersion = '4.3.1'
        junitJupiterVersion = '5.8.2'
        fglamDevkitVersion = '6.1.0-20211124'
        fglamDevkitMockVersion = '5.7.1'
        fglamDevkitBranch = '6.1.0'
        fglamVersion = '6.1.0'
    }

#### Use GeneratorTask to generate topology classes
This task relies on dev-kit installation. It will download and install it automatically. See `DevKitInstaller` class

    tasks.register('generator', com.quest.foglight.fglam.gradle.task.GenerateTask).configure {
        group 'build'
        description 'Generate topology classes, callbacks, agent.manifest, etc. based on agent definitions'
        dependsOn tasks.copyDependencies
        inputs.dir("${project.buildDir}/downloaded/fglam") // devkit files
        outputs.dir("${project.buildDir}/tooling") // generated sources

        agentDefinition = file('agent-definition.xml')
        runAllGenerators = true
        
        // Additional properties (Not sure AgentCompiler really relies on them, yet they are there for backward cmpatibiliy: GeneratorTask.*

        toolingDirectory = dir("${project.buildDir}/tooling")

        doFirst {
            println "Generating sources and agent.manifest"
        }
    }

#### Use GartridgeTask to create .gar file
    tasks.register('createGartridge', com.quest.foglight.fglam.gradle.task.GartridgeTask).configure {
        group 'build'
        description 'Creates .gar file for this project'
        dependsOn tasks.jar
        inputs.file(tasks.jar.archiveFile)
        outputs.file("${project.buildDir}/gar/${garName}.gar")

        garName = 'DockerSwarmAgent'
        agentManifest = "${project.buildDir}/tooling/agent.manifest"
        agentLibDir = "${project.buildDir}/libs"
        outputGarFile = file("${project.buildDir}/gar/${garName}.gar")

        additionalContents() {
            // If you have more filesets to add to the gar file
        }

        doFirst {
            println "Building the ${project.ext.getProperty('cartridgeName')}.gar BuildId ${foglight.buildId}"
        }
    }

### Some auto-settings
* Java `sourceCompatibility` and `targetCompatibility` are set to `JavaVersion.VERSION_1_8`
* Project is configured to create sources jar as well
* Both jar and sources jar will have meaningful information in their manifest files. Jar's manifest will also include Class-Path.
* There will be a `copyDependencies` task which copies runtimeClasspath of the project to `project.buildDir/fglam.dependenciesOutDir`
* Common dependencies will be declared for the project:
  * `implementation`: log4j-api and log4j-core
  * `compileOnly`: glueapi, gluecore, gluecommon, gluetools, glueapimockimpl
  * `testImplementation`: quest-common, quest-common-tools, junit, mockito-core
* There will be a `generatePermissions` task which looks for .PERMISSIONS files under resources folder. In case there are, it will generate FGLAM_PERMISSIONS.MF file under META-INF, and it will be done automatically, cause we configure `compileJava` task to depend on `generatePermissions`.
