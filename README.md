# Inter-Job-PoC

**Description**

This is a PoC of inter-job input injection based on a real-world example. **Note**, we **did not** launch an attack on the vulnerable repository. We used **our own accounts** to create new repositories (i.e., dreamli0 and ciplugins-poc), and mimicked the CI configuration file of the vulnerable repository for the inter-job input injection testing.

This CI configuration file includes two job, *info* and *output*. The job *info* has a benign action *mod_id* and a malicious action *action_2*. The benign action *mod_id* generates an output, which is passed to the *output* job for forming a command to run. The malicious action *action_2* now changes its own pre-step code files (e.g., by way of version reuse) to inject code to the *mod_id* action to rewrite the output of *mod_id*. The rewritten output is passed to the *output* job, when the *output* job is executed, the attack is triggered.

 
**Detailed Steps**

Below are the steps for reproducing an inter-job code injection.

- **Step 1.** The owner of the *ciplugins-poc/my-action* changes its pre-step code file (i.e., *setup.js*) by the way of version reuse (i.e., delete the verion v1.0.1 and recreate the version v1.0.1).

  Here is the changed content.
  ```
  fs.writeFileSync("/home/runner/work/_actions/dreamli0/my-action/main/index.js", 'const core = require(\'@actions/core\');core.setOutput("value", \'\";id;echo \"\');');
  ```
  This line code rewrites the benign action's code file (i.e., *index.js*).

- **Step 2.** The CI task is triggered, since the pre-step is executed before the *mod_id* action, therefore, the *index.js* of *mod_id* is modified to
  ```
  const core = require(\'@actions/core\');core.setOutput("value", \'\";id;echo \"\');
  ```
  When the action *mod_id* is executed, the output value is set to **";id;echo "**.

- **Step 3.** When the job *info* finishes, the output value is passed to the job *output* through
  ```
  mod_id: ${{steps.mod_id.outputs.value }}
  ```
  Thus, the output **";id;echo "** is passed to the job *output*.

- **Step 4.** The outputs from the job *info* are inject into the command
  ```
  echo "${{needs.info.outputs.mod_id}} version ${{needs.info.outputs.version}};"
  ```
  The value of *${{needs.info.outputs.mod_id}}* is **";id;echo "**, the value of *${{needs.info.outputs.version}}* is **main**.

  Therefore, the command is
  ```
  echo "";id;echo " version main"
  ```
  The malicious command **id** is triggered.
  
