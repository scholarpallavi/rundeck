% SSH Plugins

Rundeck by default uses SSH to execute commands on remote nodes, SCP to copy scripts to remote nodes, and locally executes commands and scripts for the local (server) node.

The SSH plugin expects each node definition to have the following properties in order to create the SSH connection:

* `hostname`: the hostname of the remote node.  It can be in the format "hostname:port" to indicate that a non-default port should be used. The default port is 22.
* `username`: the username to connect to the remote node.

When a Script is executed on a remote node, it is copied over via SCP first, and then executed.  In addition to the SSH connection properties above, these node attributes
can be configured for SCP:

* `file-copy-destination-dir`: The directory on the remote node to copy the script file to before executing it. The default value is `C:/WINDOWS/TEMP/` on Windows nodes, and `/tmp` for other nodes.
* `osFamily`: specify "windows" for windows nodes.

In addition, for both SSH and SCP, you must either configure a public/private keypair for the remote node or configure the node for SSH Password authentication.


Out of the box typical node configuration to make use of these is simple. 

* Set the `hostname` attribute for the nodes.  It can be in the format "hostname:port" to indicate that a non-default port should be used. The default port is 22.
* Set the `username` attribute for the nodes to the username to connect to the remote node.
* set up public/private key authentication from the Rundeck server to the nodes

This will allow remote command and script execution on the nodes.

See below for more configuration options.

**Sudo Password Authentication**

The SSH plugin also includes support for a secondary Sudo Password Authentication. This simulates a user writing a password to the terminal into a password prompt when invoking a "sudo" command that requires password authentication.

### SCP File Copier

In addition to the general SSH configuration mentioned for in this section, some additional configuration can be done for SCP. 

When a Script is executed on a remote node, it is copied over via SCP first, and then executed.  In addition to the SSH connection properties, these node attributes
can be configured for SCP:

* `file-copy-destination-dir`: The directory on the remote node to copy the script file to before executing it. The default value is `C:/WINDOWS/TEMP/` on Windows nodes, and `/tmp` for other nodes.
* `osFamily`: specify "windows" for windows nodes.

###  Authentication types

SSH authentication can be done in two ways, via password or public/private key.

By default, public/private key is used, but this can be changed on a node, project, or framework scope.

The mechanism used is determined by the `ssh-authentication` property.  This property can have two different values:

* `password`
* `privateKey` (default)

When connecting to a particular Node, this sequence is used to determine the correct authentication mechanism:

1. **Node level**: `ssh-authentication` attribute on the Node. Applies only to the target node.
2. **Project level**: `project.ssh-authentication` property in `project.properties`.  Applies to any project node by default.
3. **Rundeck level**: `framework.ssh-authentication` property in `framework.properties`. Applies to all projects by default.

If none of those values are set, then the default public/private key authentication is used.

### Specifying SSH Username

The username used to connect via SSH is taken from the `username` Node attribute:

* `username="user1"`

This value can also include a property reference if you want to dynamically change it, for example to the name of the current Rundeck user, or the username submitted as a Job Option value:

* `${job.username}` - uses the username of the user executing the Rundeck execution.
* `${option.someUsername}` - uses the value of a job option named "someUsername".

If the `username` node attribute is not set, then the static value provided via project or framework configuration is used. The username for a node is determined by looking for a value in this order:

1. **Node level**: `username` node attribute. Can contain property references to dynamically set it from Option or Execution values.
2. **Project level**: `project.ssh.user` property in `project.properties` file for the project.
3. **Rundeck level**: `framework.ssh.user` property in `framework.properties` file for the Rundeck installation.

### SSH private keys

The default authentication mechanism is public/private key.

The built-in SSH connector allows the private key to be specified in several different ways.  You can configure it per-node, per-project, or per-Rundeck instance.

When connecting to the remote node, Rundeck will look for a property/attribute specifying the location of the private key file, in this order, with the first match having precedence:

1. **Node level**: `ssh-keypath` attribute on the Node. Applies only to the target node.
2. **Project level**: `project.ssh-keypath` property in `project.properties`.  Applies to any project node by default.
3. **Rundeck level**: `framework.ssh-keypath` property in `framework.properties`. Applies to all projects by default.

If you private key is encrypted with a passphrase, then you can use a "Secure Option" to prompt the user to enter the passphrase when executing on the Node.  See below.

### SSH Private Key Passphrase

Using a passphrase for privateKey authentication works in the following way:

* A Job must be defined specifying a Secure Option to prompt the user for the key's passphrase.
* Target nodes must be configured to use privateKey authentication.
* When the user executes the Job, they are prompted for the key's passphrase.  The Secure Option value for the passphrase is not stored in the database, and is used only for that execution.

Therefore Private Key Passphrase authentication has several requirements and some limitations:

1. Private Key-authenticated nodes requiring passphrases can only be executed on via a defined Job, not via Ad-hoc commands (yet).
2. Each Job that will execute on such Nodes must define a Secure Option to prompt the user for the key's passphrase before execution.
3. All Nodes using passphrase protected private keys for a Job must have a matching Secure Option defined, or may use the same option name (or the default) if they share the key's passphrase (e.g. using the same private key).

Passphrases are input either via the GUI or arguments to the job if executed via CLI or API.

To enable SSH Private Key authentication, first make sure the `ssh-authentication` value is set ([#authentication-types](#authentication-types).  Second, configure the path to the private key file ([#ssh-private-keys](#ssh-private-keys)).

Next, configure a Job, and include an Option definition where `secureInput` is set to `true`.  The name of this option can be anything you want, but the default value of `sshKeyPassphrase` assumed by the node configuration is easiest.

If the value is not `sshKeyPassphrase`, then make sure to set the following attribute on each Node for password authentication:

* `ssh-key-passphrase-option` = "`option.NAME`" where NAME is the name of the Job's secure option.

An example Node and Job option configuration are below:

~~~~~~~ {.xml .numberLines}
<node name="egon" description="egon" osFamily="unix"
    username="rundeck"
    hostname="egon"
    ssh-keypath="/path/to/privatekey_rsa"
    ssh-authentication="privateKey"
    ssh-password-option="option.sshKeyPassphrase" />
~~~~~~~~~~


Job:

~~~~~~~ {.xml .numberLines}
<joblist>
    <job>
        <!-- ... -->
        <context>
          <project>project</project>
          <options>
            <option required='true' name='sshKeyPassphrase' secure='true'
              description="Passphrase for SSH Private Key"/>
          </options>
        </context>
        <!-- ... -->
    </job>
</joblist>
~~~~~~~~~

### SSH Password Authentication

Password authentication works in the following way:

* A Job must be defined specifying a Secure Option to prompt the user for the password
* Target nodes must be configured for password authentication
* When the user executes the Job, they are prompted for the password.  The Secure Option value for the password is not stored in the database, and is used only for that execution.

Therefore Password authentication has several requirements and some limitations:

1. Password-authenticated nodes can only be executed on via a defined Job, not via Ad-hoc commands (yet).
2. Each Job that will execute on password-authenticated Nodes must define a Secure Option to prompt the user for the password before execution.
3. All Nodes using password authentication for a Job must have an equivalent Secure Option defined, or may use the same option name (or the default) if they share authentication passwords.

Passwords for the nodes are input either via the GUI or arguments to the job if executed via CLI or API.

To enable SSH Password authentication, first make sure the `ssh-authentication` value is set as described in [Authentication types](#authentication-types).

Next, configure a Job, and include an Option definition where `secureInput` is set to `true`.  The name of this option can be anything you want, but the default value of `sshPassword` assumed by the node configuration is easiest.

If the value is not `sshPassword`, then make sure to set the following attribute on each Node for password authentication:

* `ssh-password-option` = "`option.NAME`" where NAME is the name of the Job's secure option.

An example Node and Job option configuration are below:

~~~~~~~~~ {.xml .numberLines}
<node name="egon" description="egon" osFamily="unix"
    username="rundeck"
    hostname="egon"
    ssh-authentication="password"
    ssh-password-option="option.sshPassword1" />
~~~~~~~~~~~~~

Job:

~~~~~~ {.xml .numberLines}
<joblist>
    <job>
        <!-- ... -->
        <context>
          <project>project</project>
          <options>
            <option required='true' name='sshPassword1' secure='true' />
          </options>
        </context>
        <!-- ... -->
    </job>
</joblist>
~~~~~~~~

### Secondary Sudo Password Authentication

The SSH provider supports a secondary authentication mechanism: Sudo password authentication.  This is useful if your security requirements are such that you require the SSH connection to be under a specific user's account instead of a generic "rundeck" account, and you still need to allow "sudo" level commands to be executed requiring a password to be entered.

This works in the following way:

* On Job execution, the user is prompted to enter a Sudo password
* After connecting to the remote node via SSH, a command requiring "sudo" authentication is issued, such as "sudo -u otheruser /sbin/some-command"
* The remote node will prompt for a sudo password, expecting user input
* The SSH Provider will write the password to the remote node
* The sudo command will execute as if a user had entered the command

Similarly to SSH Password authentication, Sudo Password Authentication requires:

* A Job must be defined specifying a Secure Option to prompt the user for the password
* Target nodes must be configured for Sudo authentication
* When the user executes the Job, they are prompted for the password.  The Secure Option value for the password is not stored in the database, and is used only for that execution.

Therefore Sudo Password Authentication has several requirements and some limitations:

1. Sudo Password authenticated nodes can only be executed on via a defined Job, not via Ad-hoc commands (yet).
2. Each Job that will execute on Sudo Password Authenticated Nodes must define a Secure Option to prompt the user for the Sudo password before execution.
3. All Nodes using Sudo password authentication for a Job must have an equivalent Secure Option defined, or may use the same option name (or the default) if they share sudo authentication passwords.

Passwords for the nodes are input either via the GUI or arguments to the job if executed via CLI or API.

To enable Sudo Password Authentication, set the `sudo-command-enabled` property/attribute to `true`.

You can configure the way the Sudo Password Authentication works by setting these properties at the Node, Project or Rundeck scopes. Simply set the attribute name on a Node, the `project.NAME` in project.properties, or `framework.NAME` in framework.properties:
 
* `sudo-command-enabled` - set to "true" to enable Sudo Password Authentication.
* `sudo-command-pattern` - a regular expression to detect when a command execution should expect to require Sudo authentication. Default pattern is `^sudo$`.
* `sudo-password-option` - an option reference ("option.NAME") to define which secure option value to use as password.  The default is `option.sudoPassword`.
* `sudo-prompt-pattern` - a regular expression to detect the password prompt for the Sudo authentication. The default pattern is `^\[sudo\] password for .+: .*`
* `sudo-failure-pattern` - a regular expression to detect the password failure response.  The default pattern is `^.*try again.*`.
* `sudo-prompt-max-lines` - maximum lines to read when expecting the password prompt. (default: `12`).
* `sudo-prompt-max-timeout` - maximum milliseconds to wait for input when expecting the password prompt. (default `5000`)
* `sudo-response-max-lines` - maximum lines to read when looking for failure response. (default: `2`).
* `sudo-response-max-timeout` - maximum milliseconds to wait for response when detecting the failure response. (default `5000`)
* `sudo-fail-on-prompt-max-lines` - true/false. If true, fail execution if max lines are reached looking for password prompt. (default: `false`)
* `sudo-success-on-prompt-threshold` - true/false. If true, succeed (without writing password), if the input max lines are reached without detecting password prompt. (default: `true`).
* `sudo-fail-on-prompt-timeout` - true/false. If true, fail execution if timeout reached looking for password prompt. (default: `true`)
* `sudo-fail-on-response-timeout` - true/false. If true, fail on timeout looking for failure message. (default: `false`)

Note: the default values have been set for the unix "sudo" command, but can be overridden if you need to customize the interaction.

Next, configure a Job, and include an Option definition where `secureInput` is set to `true`.  The name of this option can be anything you want, but the default value of `sudoPassword` recognized by the plugin can be used.

If the value is not `sudoPassword`, then make sure to set the following attribute on each Node for password authentication:

* `sudo-password-option` = "`option.NAME`" where NAME is the name of the Job's secure option.

An example Node and Job option configuration are below:

~~~~~~~~ {.xml .numberLines}
<node name="egon" description="egon" osFamily="unix"
    username="rundeck"
    hostname="egon"
    sudo-command-enabled="true"
    sudo-password-option="option.sudoPassword2" />
~~~~~~~~~~~~

Job:

~~~~~~~~~~ {.xml .numberLines}
<joblist>
    <job>
         <sequence keepgoing='false' strategy='node-first'>
          <command>
            <exec>sudo apachectl restart</exec>
          </command>
        </sequence>

        <context>
          <project>project</project>
          <options>
            <option required='true' name='sudoPassword2' secure='true' 
                    description="Sudo authentication password"/>
          </options>
        </context>
        ...
    </job>
</joblist>
~~~~~~~~~~

### Multiple Sudo Password Authentication

You can enable a further level of sudo password support for a node.  If you have
the requirement of executing a chain of "sudo" commands, such as "sudo -u user1
sudo -u user2 command", and need to enable password input for both levels of
sudo.  This is possible by configuring a secondary set of properties for your
node/project/framework.

The configuration properties are the same as those for the first-level of sudo
password authentication described in [Configuring Secondary Sudo Password
Authentication](#secondary-sudo-password-authentication), but with a
prefix of "sudo2-" instead of "sudo-", such as:

    sudo2-command-enabled="true"
    sudo2-command-pattern="^sudo .+? sudo .*$"

This would turn on a mechanism to expect and respond to another sudo password
prompt when the command matches the given pattern.

If a value for "sudo2-password-option" is not set, then a default value of
`option.sudo2Password` will be used.

**A note about the "sudo2-command-pattern":**

The sudo authentication mechanism uses two regular expressions to test whether it should be 
invoked.

For the first sudo authentication, the "sudo-command-pattern" value is matched against
the **first component of the command being executed**. The default value for this pattern is `^sudo$`.
So a command like "sudo -u user1 some command" will match correctly.  You can modify the 
regular expression (e.g. to support "su"), but it will always only match against the first 
part of the command.

If "sudo2-command-enabled" is "true", then the "sudo2-command-pattern" is also checked 
and if it matches then another sudo authentication is enabled.
However this regular expression is tested against the **entire command string**
to make it possible to determine whether it should be enabled. The default value is 
`^sudo .+? sudo .*$`. If necessary you should customize the value.



## SSH System Configuration

* The SSH configuration requires that the Rundeck server machine can
  ssh commands to the client machines. 
* SSH is assumed to be installed and configured appropriately to allow
  this access.   
* SSH can be configured for either *password* based authentication or *public/private key* based authentication.
* For public/private key authentication:
    * There are many resources
available on how to configure ssh to use public key authentication
instead of passwords such as:
[Password-less logins with OpenSSH](http://www.debian-administration.org/articles/152) or [How-To: Password-less SSH](http://www.cs.wustl.edu/~mdeters/how-to/ssh/).
    * If your private key file has a passphrase, each Job definition that will execute on the node must be configured correctly.
* For password authentication:
    * each Node definition must be configured to allow password authentication
    * each Job definition that will use it must be configured correctly


### SSH key generation

* The Rundeck installation can be configured to use RSA _or_ DSA
  type keys.
  
Here's an example of SSH RSA key generation on a Linux system:

    $ ssh-keygen -t rsa
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/demo/.ssh/id_rsa): 
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Your identification has been saved in /home/demo/.ssh/id_rsa.
    Your public key has been saved in /home/demo/.ssh/id_rsa.pub.
    The key fingerprint is:
    a7:31:01:ca:f0:62:42:9d:ab:c8:b7:9c:d1:80:76:c6 demo@ubuntu
    The key's randomart image is:
    +--[ RSA 2048]----+
    | .o . .          |
    |.  * . .         |
    |. = =   .        |
    | = E     .       |
    |+ + o   S .      |
    |.o o .   =       |
    |  o +   .        |
    |   +             |
    |                 |
    +-----------------+

### Configuring remote machine for SSH 

To be able to directly ssh to remote machines, the SSH public key of
the client should be shared to the remote machine.
  
Follow the steps given below to enable ssh to remote machines.

The ssh public key should be copied to the `authorized_keys` file of
the remote machine. The public key will be available in
`~/.ssh/id_rsa.pub` file.
  
The `authorized_keys` file should be created in the `.ssh` directory of
the remote machine.
  
The file permission of the authorized key should be read/write for
the user and nothing for group and others. To do this check the
permission and change it as shown below.

    $ cd ~/.ssh
    $ ls -la
    -rw-r--r--   1 raj  staff     0 Nov 22 18:14 authorized_keys

    $ chmod 600 authorized_keys 
    $ ls -la
    -rw-------   1 raj  staff     0 Nov 22 18:14 authorized_keys

The permission for the .ssh directory of the remote machine should
be read/write/execute for the user and nothing for the group and
others. To do this, check the permission and change it as shown
below.  

    $ ls -la
    drwxr-xr-x   2 raj  staff    68 Nov 22 18:19 .ssh
    $ chmod 700 .ssh
    $ ls -la
    drwx------   2 raj  staff    68 Nov 22 18:19 .ssh

If you are running Rundeck on Windows, we heartily recommend using
[Cygwin] on Windows as it includes SSH and a number of
Unix-like tools that are useful when you work in a command line
environment.

[Cygwin]: http://www.cygwin.org


### Passing environment variables through remote command

To pass environment variables through remote command
dispatches, it is required to properly configure the SSH server on the
remote end. See the `AcceptEnv` directive in the "sshd\_config(5)"
manual page for instructions. 

Use a wild card pattern to permit `RD_` prefixed variables to provide
open access to Rundeck generated environment variables.

Example in sshd_config:

    # pass Rundeck variables
    AcceptEnv RD_*

[SSH]: http://en.wikipedia.org/wiki/Secure_Shell
