fimport java.text.SimpleDateFormat;
import java.util.Date;

pipeline {
    agent any
    tools {
        maven "maven3"
    }
    environment {
        MVN = "mvn --settings ~/mvn-settings-xml/settings.xml"
    }
    stages {
        stage("Calculate Version Number") {
            steps {
                milestone(1)
                script {
                    sshagent([env.GIT_SSH_CREDENTIALS_ID]) {
                        fetchAllGitTags(this)
                        env.VERSION = getNextVersion(this, "1.0", env.BRANCH_NAME)
                        println("Building with version " + env.VERSION)
                        currentBuild.description = env.VERSION
                    }
                }
            }
        }
        stage("Check Allowed Branch") {
            steps {
                script {
                    if (! (env.BRANCH_NAME == "master" || env.BRANCH_NAME.contains("release"))) {
                        currentBuild.description = "Skipped, because running from branches is not allowed"
                        currentBuild.getRawBuild().getExecutor().interrupt(Result.SUCCESS)
                    }
                }
            }
        }
        stage("Set and update Version") {
            // current GCP setup is not backed by an NFS, but a simple PVC instead,
            // which can be used by a single pod only
            // because of this, we allow only a single build to run at any time
            steps {
                milestone(2)
                lock(resource: "MavenCacheSingleUserAllowed", inversePrecedence: true) {
                    sh "$MVN versions:display-dependency-updates versions:use-latest-versions versions:update-properties -U"
                    sh "$MVN versions:set -DnewVersion=${env.VERSION}"
                }
            }
        }
        stage("Maven Deploy") {
            steps {
                milestone(3)
                lock(resource: "MavenCacheSingleUserAllowed", inversePrecedence: true) {
                    sh "$MVN clean deploy"
                }
            }
        }
        stage("Git Tag") {
            steps {
                milestone(4)
                sshagent([env.GIT_SSH_CREDENTIALS_ID]) {
                    sh "git config --global user.email 'jenkins@poeschmann.com'"
                    sh "git config --global user.name 'jenkins'"
                    sh "git add pom.xml"
                    sh "git commit -m 'Automatic BOM IMPL update'"
                    sh "git pull --rebase origin master"
                    sh "git push origin HEAD:master"
                    sh "git tag ${env.VERSION}"
                    sh "git push origin ${env.VERSION}"
                }
            }
        }
    }
}

/**
 * Generate the next version from history - for master or release branches,
 * this will be based upon tags - for all other branches, it will be a
 * snapshot with the current date.
 */
private static String getNextVersion(Script script, String majorVersion, String branchName) {
    if (branchName == "master" || branchName.contains("release")) {
        Integer currentVersion = getLatestMinorVersionFromTags(script, majorVersion);
        int nextVersion = currentVersion != null ? currentVersion + 1 : 0;
        return majorVersion + "." + nextVersion
    } else {
        return "1.0-" + new SimpleDateFormat("yyyy-dd-MM-HH-mm-ss-SSS").format(new Date()) + "-SNAPSHOT"
    }
}

/**
 * Return the latest minor version for the given major version from the tags, or NULL.
 */
private static Integer getLatestMinorVersionFromTags(Script script, String majorVersion) {
    String gitTagsRaw = script.sh(returnStdout:true, script: "git tag -l --sort=-v:refname ${majorVersion}.*")
    String [] tags = gitTagsRaw != null ? gitTagsRaw.split("\n") : null
    if (tags == null || tags.size() == 0 || !(tags[0]?.trim())) {
        return null;
    } else {
        String gitTag = tags[0]
        // NOT checking for new versions here, since this is the BOM! Allow any
        // new versions
        return getLatestMinorVersionFromTag(gitTag);
    }
}

/**
 * Cause git to download the tags.
 */
private static void fetchAllGitTags(Script script) {
    script.sh(returnStdout:true, script: "git fetch --all --tags")
}

/**
 * Extract the minor version out of the given tag (i.e. 1.234 -> 234
 * or 1.2.3.4 -> 4)
 */
private static String getLatestMinorVersionFromTag(String gitTag) {
    String[] tagComponents = gitTag.split("\\.");
    return tagComponents[tagComponents.length-1] as Integer
}