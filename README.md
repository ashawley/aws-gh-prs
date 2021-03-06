[AWS console]: http://console.aws.amazon.com
[JGit]: https://eclipse.org/jgit/
[JSch]: http://www.jcraft.com/jsch/
[github-api]: https://github.com/code-check/github-api-scala
[Async HTTP Client]: https://github.com/AsyncHttpClient/async-http-client
[json4s]: http://json4s.org/
[AWS Lambda Java libraries]: https://github.com/aws/aws-lambda-java-libs
[SBT Assembly]: http://github.com/sbt/sbt-assembly
[SBT]: http://scala-sbt.org
[SLF4J]: https://www.slf4j.org/
[Typesafe Scala Logging]: https://github.com/typesafehub/scala-logging
[knobs]: http://verizon.github.io/knobs/
[Typesafe Config]: https://typesafehub.github.io/config/
[list the hooks]: https://developer.github.com/v3/repos/hooks/#list-hooks
[edit the hook]: https://developer.github.com/v3/repos/hooks/#edit-a-hook

## Merging pull requests on AWS Lambda with GitHub webhooks

The following Scala code builds a Java JAR file that can run on AWS
Lambda to automatically integrate pull requests for a project.

### Overview

A Scala program that is tightly coupled to the following libraries:

- The Java library, [JGit], by Eclipse can clone Git repositories,
checkout branches, merge them and push them.
- The Java library, [JSch], from JCraft provides SSH support for JGit
including support for user and host key verification.
- Use of the GitHub API is with a Scala library, [github-api], by SHUNJI
Konishi of Codecheck.
- Connect to GitHub's API over HTTP with a Java library, [Async HTTP
Client], by Ning.
- JSON serialization with the Scala library, [json4s], by Ivan Porto
Carrero and KAZUHIRO Sera.
- The [AWS Lambda Java libraries] by Amazon AWS provide the ability to
run the JAR in their *serverless* Java 8 runtime.
- JAR produced using the SBT plugin, [SBT Assembly], by Eugene Yokota.
- JVM file system housekeeping provided by the sbt.io Scala library,
from [SBT] team at Lightbend, Inc.
- Configuration file support managed by [knobs] from Verizon and
[Typesafe Config] from Lightbend, Inc.
- Logging provided by [SLF4J] of QoS.ch and [Typesafe Scala Logging] by
Lightbend, Inc.

Steps in detail:

- Receive event from GitHub by way of AWS API Gateway request
- Load config file from `application.conf`
- Verify repo in request is same repo in config file
- Find name of base branch in config file to merge on to
- Use GitHub token in config file to call GitHub API
- Find all pull requests in the repo for the base branch
- Set HEAD commit to all pull requests to pending
- Checkout base branch with Git and SSH
- Use SSH keys and known_hosts specified in config file
- Use SSH keys and known_hosts included in JAR file
- Create integration branch specified in config file with Git
- Merge pull requests on to integration branch with Git
- Force push to GitHub using Git and SSH
- If merge succeeds, merged branches
- If merge fails, don't push, and return unmergable branch
- Notify success or failure with GitHub status API

### Testing locally

This code is designed to run at AWS Lambda.  This makes it difficult
to test outside of AWS Lambda.  The app runs in Lambda with input from
API gateway based on GitHub webhooks and is called in AWS Lambda
in the peculiar way that Lambda operates.

The `LambdaApp` trait is defined to enforce a type signature for an
AWS Lambda functions, and can help with testing locally by providing
mock data inputs to run against.  However, certain activities are not
mocked, but instead operate against live systems: The application will
request data from GitHub's API and conduct Git operations against the
Git repo configured in `application.conf`.  Currently, these services
are not mocked in any tests.

Some of the steps for testing locally are the same as getting the
application to run in AWS Lambda.

- Rename the `application.conf.template` file in `src/main/resources`
to `application.conf`.  - Generate an SSH key with an empty passphrase

```bash
$ ssh-keygen -t rsa -f src/main/resources/ssh/id_rsa
```

- With the files `id_rsa` and `id_rsa.pub` in the `src/main/resources/ssh`
directory.  Specify the names of these files in the `application.conf`
file.

- Specify the name of the Git repository and the Git branch you want to
update with auto-merged PRs in the `application.conf` file.

- Add the public key to ~~the repository~~ your user account in GitHub
as ~~a deploy~~ an SSH key ~~with *write access*~~

 - Visit ~~http://github.com/my/repo/settings/keys~~
https://github.com/settings/ssh

- Create a GitHub access token in your user settings at

 - http://github.com/settings/tokens/new

 - Click on **Generate new token**

 - Provide a **Token description**, such as **AWS Lambda**

 - Select the **repo** scope for access

 - Click **Generate token**

 - Copy the personal access token

- Then type `run` in SBT

```
> run
[info] Running prs.Main
START RequestId: dc8beb69-1858-41f0-884c-9058930ea98f Version: $LATEST
22:07:26.356 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:105 - Starting aws-gh-prs 0.1-SNAPSHOT
22:07:26.375 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:107 - Inspecting API Gateway notification...
22:07:26.695 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:116 - Processed 1 event(s)
22:07:26.934 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:139 - Pull request was organization/master..username/hotfix1
22:07:26.935 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:180 - Querying GitHub for open pull request(s)...
22:07:32.775 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:202 - Found 2 pull request(s)
22:07:32.794 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:220 - Working directory is /tmp/sbt_39882bf2
22:07:32.794 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:222 - Cloning git@github.com:organization/example.git...
22:08:57.255 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:235 - Configuring repository...
22:08:57.256 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:242 - Set user to User Name
22:08:57.256 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:243 - Set email to user@example.com
22:08:57.256 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:245 - Configuring remotes...
22:08:57.714 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:263 - Remote username at git@github.com:username/example.git
22:08:57.733 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:274 - Configured 1 remote(s)
22:09:07.413 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:281 - Fetching username:example...
22:09:11.893 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:294 - Checking out organization/master...
22:09:13.093 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:302 - Creating branch staging...
22:09:13.716 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:313 - Merging username/feature1...
22:09:15.654 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:313 - Merging username/hotfix1...
22:09:22.554 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:361 - Pushing staging...
22:09:27.533 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:370 - Setting status on pull request(s)...
22:09:27.556 <dc8beb69-1858-41f0-884c-9058930ea98f> [main] INFO prs.Main$:383 - Success status set to pull request(s)
END RequestId: dc8beb69-1858-41f0-884c-9058930ea98f
REPORT RequestId: dc8beb69-1858-41f0-884c-9058930ea98f	Duration: 126355.07 ms	Billed Duration: 126400 ms Memory Size: 2047 MB  Max Memory Used: 518 MB
[
  "Merged branch(es) 'username/feature1', 'username/hotfix1' into 'staging'"
]
[success] Total time: 129 s, completed Mar 21, 2017 6:51:35 PM
```

### Building the JAR

To run this app on AWS Lambda, run the task provided by [SBT
Assembly], to build a JAR file.

```
> assembly
[info] Including: typesafe_2.12-5.0.32.jar
[info] Including: joda-convert-2.2.0.jar
[info] Including: json4s-ext_2.12-3.6.6.jar
[info] Including: jackson-databind-2.9.8.jar
[info] Including: io_2.12-1.2.2.jar
[info] Including: aws-lambda-java-events-2.2.7.jar
[info] Including: log4j-1.2.17.jar
[info] Including: jna-4.5.0.jar
[info] Including: jzlib-1.1.1.jar
[info] Including: json4s-jackson_2.12-3.6.6.jar
[info] Including: paranamer-2.8.jar
[info] Including: scala-reflect-2.12.10.jar
[info] Including: jackson-core-2.9.8.jar
[info] Including: jsch-0.1.54.jar
[info] Including: github-api_2.12-0.3.0-SNAPSHOT.jar
[info] Including: parser_2.12-0.5.8.jar
[info] Including: jackson-annotations-2.9.0.jar
[info] Including: httpcore-4.4.4.jar
[info] Including: scopt_2.12-3.7.1.jar
[info] Including: aws-lambda-java-core-1.2.0.jar
[info] Including: slf4j-api-1.7.28.jar
[info] Including: aws-lambda-java-log4j-1.0.0.jar
[info] Including: scala-logging_2.12-3.8.0.jar
[info] Including: json4s-core_2.12-3.6.7.jar
[info] Including: httpclient-4.5.2.jar
[info] Including: json4s-scalap_2.12-3.6.7.jar
[info] Including: slf4j-log4j12-1.7.28.jar
[info] Including: async-http-client-1.9.40.jar
[info] Including: jna-platform-4.5.0.jar
[info] Including: json4s-ast_2.12-3.6.7.jar
[info] Including: org.eclipse.jgit-4.11.9.201909030838-r.jar
[info] Including: joda-time-2.10.1.jar
[info] Including: commons-logging-1.2.jar
[info] Including: json4s-native_2.12-3.6.7.jar
[info] Including: netty-3.10.6.Final.jar
[info] Including: config-1.2.1.jar
[info] Including: JavaEWAH-1.1.6.jar
[info] Including: scala-library-2.12.10.jar
[info] Including: apple-file-events-1.3.2.jar
[info] Including: machinist_2.12-0.6.2.jar
[info] Including: fs2-core_2.12-0.10.0-M10.jar
[info] Including: core_2.12-5.0.32.jar
[info] Including: cats-kernel_2.12-1.0.0-RC2.jar
[info] Including: cats-free_2.12-1.0.0-RC2.jar
[info] Including: commons-codec-1.9.jar
[info] Checking every *.class/*.jar file's SHA-1.
[info] Merging files...
[warn] Merging 'LICENSE' with strategy 'first'
[warn] Merging 'META-INF/DEPENDENCIES' with strategy 'discard'
[warn] Merging 'META-INF/ECLIPSE_.RSA' with strategy 'discard'
[warn] Merging 'META-INF/ECLIPSE_.SF' with strategy 'discard'
[warn] Merging 'META-INF/LICENSE' with strategy 'discard'
[warn] Merging 'META-INF/LICENSE.txt' with strategy 'discard'
[warn] Merging 'META-INF/MANIFEST.MF' with strategy 'discard'
[warn] Merging 'META-INF/NOTICE' with strategy 'discard'
[warn] Merging 'META-INF/NOTICE.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.base64.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.bouncycastle.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.commons-logging.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.felix.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.jboss-logging.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.jsr166y.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.jzlib.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.log4j.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.protobuf.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.slf4j.txt' with strategy 'discard'
[warn] Merging 'META-INF/license/LICENSE.webbit.txt' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-core/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-core/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-events/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-events/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-log4j/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.amazonaws/aws-lambda-java-log4j/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-annotations/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-annotations/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-core/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-core/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-databind/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.fasterxml.jackson.core/jackson-databind/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.googlecode.javaewah/JavaEWAH/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.googlecode.javaewah/JavaEWAH/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.jcraft/jsch/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.jcraft/jsch/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.jcraft/jzlib/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.jcraft/jzlib/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.ning/async-http-client/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.ning/async-http-client/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.thoughtworks.paranamer/paranamer/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/com.thoughtworks.paranamer/paranamer/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/commons-codec/commons-codec/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/commons-codec/commons-codec/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/commons-logging/commons-logging/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/commons-logging/commons-logging/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/io.netty/netty/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/io.netty/netty/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/joda-time/joda-time/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/joda-time/joda-time/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/log4j/log4j/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/log4j/log4j/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.apache.httpcomponents/httpclient/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.apache.httpcomponents/httpclient/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.apache.httpcomponents/httpcore/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.apache.httpcomponents/httpcore/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.eclipse.jgit/org.eclipse.jgit/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.eclipse.jgit/org.eclipse.jgit/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.joda/joda-convert/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.joda/joda-convert/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.slf4j/slf4j-api/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.slf4j/slf4j-api/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.slf4j/slf4j-log4j12/pom.properties' with strategy 'discard'
[warn] Merging 'META-INF/maven/org.slf4j/slf4j-log4j12/pom.xml' with strategy 'discard'
[warn] Merging 'META-INF/services/com.fasterxml.jackson.core.JsonFactory' with strategy 'discard'
[warn] Merging 'META-INF/services/com.fasterxml.jackson.core.ObjectCodec' with strategy 'discard'
[warn] Merging 'NOTICE' with strategy 'first'
[warn] Strategy 'discard' was applied to 65 files
[warn] Strategy 'first' was applied to 2 files
[info] SHA-1: c557f6485eb6fb8ec55cd9e91416b24fca59efd0
[success] Total time: 76 s (01:16), completed Sep 12, 2019 10:23:42 AM
```

### Testing at AWS

- Follow the instructions for testing locally, see above

- Create a new API Gateway

 - From the [AWS console], click on the **API Gateway** service

 - Click on **Create API**

 - Enter a API name, such as **GitHub-My-Repo-Testing**

 - Click on **Create API**

 - Click on **Actions**

 - Click on **Create Resource**, such as "Web Hook"

 - Enter a **Resource Name**, such as "webhook"

 - Click on **Create**

 - Click on **Actions**

 - Click on **Create Method**

 - Select **POST** from the drop-down

 - Click on the new **POST** entry

 - Click the checkbox for **Use Lambda Proxy integration**

 - Enter the **Lambda Function** name

 - Click on **Save**

 - Copy the **ARN** for the API Gateway


- Connect the GitHub repo to the API Gateway topic

 - In GitHub, visit your repo's integration settings

   - http://github.com/my/repo/settings/hooks

 - Click **Add webhook**

 - For **Payload URL**, enter the **ARN** from earlier

 - For **Content type** choose **application/json**

 - For **Secret**, enter the **Authorization key** from earlier

 - For the **Aws secret**, enter the **Secret access key**

 - Choose **Let me select individual events**

 - Click the checkbox for **Pushes** and **Pull requests**

 - Click **Add hook**

- Add the app to Lambda as a JAR

 - Follow the instructions below, see "Uploading to AWS Lambda"

### Uploading to AWS Lambda

- From the [AWS console], click on **Configure function** from the
Lambda service

- Click **Create a Lambda Function**

- Don't select a blue print

- Click on **Configure triggers**

- Add an **API Gateway** trigger

- Select the API Gateway you created earlier

- Click the **Enable trigger** checkbox

- Click **Next**

- Provide a **Name** for the function, such as **GithubMyRepoTest**

- Leave the description blank

- Set the **Runtime** to **Java 8**

- For **Function package**, click the **Upload**

- Upload the JAR file created by SBT assembly at
`target/scala-2.11/aws-gh-prs-assembly-0.1-SNAPSHOT.jar`

- Enter the handler as, `prs.Main::handleRequest`

- Choose **Create new role from template(s)**

- Enter **LambdaTestRole** as the role name

- Leave **Policy templates** blank

- Reduce the memory to **384 MB**, and leave the timeout at *240 seconds**

- Keep the setting for **VPC** to none

- Click **Next**

- Click **Create function**

- Click **Actions** and then **Configure test event**

- From the **Sample event template** drop-down, choose **API Gateway**

- Click **Save and Test**

- The test will succeed but only return the empty array, `[]`

- To change the test to one that is closer to a GitHub event

 - Insert a minimal string of json in the API Gateway **body**
```
    "body": "{\"action\":\"opened\",\"number\":1,\"pull_request\":{\"state\":\"open\",\"head\":{\"ref\":\"changes\",\"repo\":{\"name\":\"public-repo\",\"owner\":{\"login\":\"baxterthehacker\"}}},\"base\":{\"ref\":\"master\",\"repo\":{\"name\":\"public-repo\",\"owner\":{\"login\":\"baxterthehacker\"}}}}}"
```
 - Add to **headers** an **X-GitHub-Event**
```
    "headers": {
      [...]
      "X-GitHub-Event": "pull_request",
      [...]
    }
```

### Warranty

**Buyer beware**: This application will overwrite Git branches.
Should the merge complete successfully, the application does a forced
push to a branch of a remote Git repo.  The branch is the one that is
configured in `application.conf`.  A forced push could result in data
loss.  The SSH ~~deploy~~ user key added to GitHub will have access to
all the repos the user is configured for.  However, by the nature of
private repos in GitHub, write access to a root repo in GitHub will
provide the same access to forked repos.  This is advantageous since
the program requires read-access to forks.  Public repos provide
read-access, by default, ~~without a deploy key~~ for any GitHub user.
Since this program requires write-access to the repo configured in
`application.conf`, the ~~deploy~~ user key will have write-access to
~~forks of private~~ all repos that user has access to.

 The code will verify that the Git repo specified in
`application.conf`, including the branch to monitor for pull requests.

The application as written doesn't catch any exceptions.  There's no
need for the program to recover given it is an internal task that runs
in AWS Lambda, a serverless environment.

The SSH key used to pull from private repos or push to repositories,
is included in the JAR file.  This is a security concern, since the
SSH key will have write privileges to remote Git repositories at
GitHub.

Currently, if there is an error then it results in an exception being
thrown and execution being halted.

Some of the failure conditions that should be caught with a friendly
error message, include

- Unable to read or write to temporary directory on filesystem
- Conf file missing from JAR
- Conf file missing directives
- API Gateway json is malformed
- GitHub json is malformed
- SSH keys missing from the JAR
- SSH hosts key file, `known_hosts`, is missing from JAR
- SSH host key verification fails
- Git client is broken or unavailable
- Git server is broken or unavailable
- A Git remote doesn't exist
- Base branch doesn't exist
- HTTP is broken or unavailable
- GitHub API is broken or unavailable
- Configuring Git remotes fails
- Git fetching fails
- Checking out remote branch fails
- Git merge fails
- Git push to remote fails

### References

- https://developer.github.com/webhooks/
- http://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/
- http://aws.amazon.com/blogs/compute/dynamic-github-actions-with-aws-lambda/
- http://eclipse.org/jgit/
- http://github.com/code-check/github-api-scala/
- http://www.jcraft.com/jsch/
- http://github.com/ashawley/aws-lambda-scala-hello-world