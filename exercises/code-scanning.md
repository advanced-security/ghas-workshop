## Code scanning

Code scanning enables developers to integrate security analysis tooling into their developing workflow. In this exercise, we will focus on the CodeQL static analysis tooling that helps developers detect common vulnerabilities and coding errors.

### Contents

- [Enabling code-scanning](#enabling-code-scanning)
- [Reviewing a failed analysis job](#reviewing-a-failed-analysis-job)
- [Customizing the build process in the CodeQL workflow](#customizing-the-build-process-in-the-codeql-workflow)
- [Reviewing and managing results](#reviewing-and-managing-results)
- [Triaging a result in a PR](#triaging-a-result-in-a-pr)
- [Customizing CodeQL configuration](#customizing-codeql-configuration)
- [Adding your own code scanning suite to exclude rules](#adding-your-own-code-scanning-suite-to-exclude-rules)

### _**Practical Exercise 2**_

#### Enabling code scanning

1. Go to the `Code scanning alerts` section in the `Security` tab.

2. Start the `Set up this workflow` step in the `CodeQL Analysis` card.

3. Review the created Action workflow and accept the default proposed workflow.

4. Head over to the `Actions` tab to see the created workflow in action.

 
#### Reviewing and managing results

1. Go to the `Code scanning results` in the `Security` tab.

2. For a result, determine:
    1. The issue reported.
    1. The corresponding query id.
    1. Its `Common Weakness Enumeration` identifier.
    1. The recommendation to solve the issue.
    1. The path from the `source` to the `sink`. Where would you apply a fix?
    1. Is it a *true positive* or *false positive*?

#### Triaging a result in a PR

The default workflow configuration enables code scanning on PRs.
Follow the next steps to see it in action.

1. Add a vulnerable snippet of code and commit it to a patch branch and create a PR.

    Make the following change in `frontend/src/components/AuthorizationCallback.vue:27`

    ```javascript
     - if (this.hasCode && this.hasState) {
     + eval(this.code)    
     + if (this.hasCode && this.hasState) {
    ```
2. Is the vulnerability detected in your PR?

#### Enabling CodeQL in third-party CI/CD

CodeQL analysis is not limited to only GitHub Actions - it can also be run in other CI/CD pipelines, such as Azure DevOps or Jenkins.

To demonstrate this, we will enable CodeQL analysis in an Actions pipeline using only shell commands - no Actions!

 1. Create a copy of the `codeql-analysis.yml` file named `codeql-analysis-third-party.yml`.

 2. Remove the `init`, `autobuild` and `analyze` steps.

 3. Remove the `strategy` section to disable the matrix job. (_note: matrix jobs are not currently supported in 3rd party CI/CD integrations_)

 3. Add a step to download the CodeQL runner and make it executable:

   ```
    - name: Install codeql-runner
      run: |
        wget -q https://github.com/github/codeql-action/releases/latest/download/codeql-runner-linux
        sudo install -o root -g root -m 0755 codeql-runner-linux /usr/local/bin/codeql-runner
   ```
 4. Add a step to execute the CodeQL runner

   ```
    - name: Execute CodeQL Runner
      run: |
        echo "${{ github.token }}" | codeql-runner init \
          --github-url "https://github.com" \
          --repository "${{ github.repository }}" \
          --github-auth-stdin \
          --languages "java,javascript,python"
        codeql-runner autobuild
        echo "${{ github.token }}" | codeql-runner analyze \
          --github-url "https://github.com" \
          --repository "${{ github.repository }}" \
          --github-auth-stdin \
          --commit "${{ github.sha }}" \
          --ref "${{ github.ref }}"
   ```
#### Customizing CodeQL Configuration

By default, CodeQL uses a selection of queries that provide high quality security results.
However, you might want to change this behavior to:

- Include code-quality queries.
- Include queries with a lower signal to noise ratio to detect more potential issues.
- To exclude queries in the default pack because they generate *false positives* for your architecture.
- Include custom queries written for your project.

1.  Create the file `.github/codeql/codeql-config.yml` and enable the `security-and-quality` suite.

    **Hints**

    1. A configuration file contains a key `queries` where you can specify additional queries as follows

        ```yaml
        name: "My CodeQL config"

        queries:
            - uses: <insert your query suite>
        ```
2. Enable your custom configuration in the code scanning workflow file `.github/codeql/codeql-config.yml`

    **Hints**

    1. The `init` action supports a `config-file` parameter to specify a configuration file.

3. After the code scanning action has completed, are there new code scanning results?

#### Adding your own code scanning suite to exclude rules

The queries that are executed is determined by the code scanning suite for a target language.
You can create your own code scanning suite to change the set of included queries.

By creating our own code scanning suite, we can exclude the rule that caused the false positive in our Java project.

1. Create the file `custom-queries/code-scanning.qls` with the contents

    ```yaml
    # Reusing existing QL Pack
    - import: codeql-suites/javascript-code-scanning.qls
      from: codeql-javascript
    - import: codeql-suites/java-code-scanning.qls
      from: codeql-java
    - import: codeql-suites/python-code-scanning.qls
      from: codeql-python
    - import: codeql-suites/go-code-scanning.qls
      from: codeql-go
    - exclude:
      id:
        - <insert rule id of false positive>
    ```

2. Configure the file `.github/codeql/codeql-config.yml` to use our suite

    **Hint** We are now running both the default code scanning suite and our own custom suite.
    To prevent double results disable the default queries with the option `disable-default-queries: true`

3. After the code scanning action has completed, is the false positive still there?

<details>
<summary>Solution</summary>

```yaml
# Reusing existing QL Pack
- import: codeql-suites/javascript-code-scanning.qls
    from: codeql-javascript
- import: codeql-suites/java-code-scanning.qls
from: codeql-java
- import: codeql-suites/python-code-scanning.qls
from: codeql-python
- import: codeql-suites/go-code-scanning.qls
from: codeql-go
- exclude:
    id:
    - java/spring-disabled-csrf-protection
```
</details>

4. Try running additional queries with `security-extended` or `security-and-quality`. What kind of results do you see?

5. Try specifying directories to scan or not to scan. Why would you include this in the configuration?

#### Understanding how to add a custom query

One of the strong suites of CodeQL is its high-level language QL that can be used to write your own queries.
_If you have experience with CodeQL and have come up with your own query so far, take this time to commit those changes and see if any alerts were produced._
Regardless of experience, the next steps show you how to add one.

1. Make sure to create a QL pack file. For example, `custom-queries/go/qlpack.yml` with the contents

    ```yaml
    name: My Go queries
    version: 0.0.0
    libraryPathDependencies:
        - codeql-go
    ```

    This file creates a [QL query pack](https://help.semmle.com/codeql/codeql-cli/reference/qlpack-overview.html) used to organize query files and their dependencies.

2. Then, create the actual query file. For example, `custom-queries/go/jwt.ql` with the contents

    ```ql
    /**
    * @name Missing token verification
    * @description Missing token verification
    * @id go/user-controlled-bypass
    * @kind problem
    * @problem.severity warning
    * @precision high
    * @tags security
    */
    import go
    /*
    * Identify processors that are missing the token verification:
    *
    * func(token *jwt.Token) (interface{}, error) {
    *    // Don't forget to validate the alg is what you expect:
    *    //if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
    *    //        return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
    *    //}
    *    ...
    * }
    */
    from FuncLit f
    where
        // Identify the function via the argument part of the its signature
        //     func(token *jwt.Token) (interface{}, error) { ... }
        f.getParameter(0).getType() instanceof PointerType and
        f.getParameter(0).getType().(PointerType).getBaseType().getName() = "Token" and
        f.getParameter(0).getType().(PointerType).getBaseType().getPackage().getName() = "jwt" and
        // and check whether it uses jwt.SigningMethodHMAC in any way
        not exists(TypeExpr t |
            f.getBody().getAChild*() = t and
            t.getType().getName() = "SigningMethodHMAC" and
            t.getType().getPackage().getName() = "jwt"
        )
    select f, "This function should be using jwt.SigningMethodHMAC"
    ```
3. Then, add the query to the CodeQL configuration file `.github/codeql/codeql-config.yml`

**Hint** The `uses` key accepts repository relative paths.

<details>
<summary>Solution</summary>

```yaml
name: "My CodeQL config"

disable-default-queries: true

queries:
    - uses: security-and-quality
    - uses: ./custom-queries/code-scanning.qls
    - uses: ./custom-queries/go/jwt.ql

```
</details>

#### _Stretch Exercise 3: Adding a custom query from an external repository_

How would you incorporate that query/queries that were written on the first day of this bootcamp?

<details>
<summary>Solution</summary>

```yaml
name: "CodeQL Config"

disable-default-queries: false

queries:
  - name: go-custom-queries
    uses: advanced-security/ghas-bootcamp-march-2021-query/go/src@main
  - uses: security-and-quality
```
</details>


💡**Looks like we've made it to the end! [Click here for additional references](api-references.md).** 💡
