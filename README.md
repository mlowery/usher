# usher

(U)niversal (S)hell (H)istory - er

Note: This project is just documentation. There are no other artifacts.

Shell history is possibly the richest source of assistance in the terminal. It's arguably better than a snippets manager (no work to create a new snippet). It's arguably better than documentation (your previous commands are already customized for your environment). And it's arguably better than a clipboard manager (since you might not have copied the entire command into the clipboard).

But how does one capture history across hosts (ssh), shells (zsh, bash), and logins (root, shared logins, and personal accounts)? You could set up custom `.zshrc` or `.bashrc` files but this may not be practical given the number of hosts, shells, and logins. And you certainly cannot do this with a shared login.

## Requirements

* [iTerm2](https://iterm2.com/) (and macOS)

## How It Works

Essentially, it works like this:

1. Using an iTerm [trigger](https://www.iterm2.com/triggers.html), detect a shell prompt. Then inject some history-dumping shell code using the iTerm **Send Text** action. This history-dumping code is triggered on shell exit.
2. Using another iTerm trigger, detect the history dump. Then append the history to a file using the iTerm **Run Silent Coprocess** action.

## Installation

1. In iTerm preferences, go to **Profiles** > *profile* > **Advanced** > **Triggers** > **Edit**.
2. Click **+**.
3. The **Regular Expression** will vary but here's an example to detect a CentOS prompt:

```regexp
^\[[\w\-]+@[\w\-]+ [^\s]+\][\#,\$]
```

4. For **Action**, choose **Send Text**.
5. For **Parameters**, below is an example for Bash. Be careful that whatever you put in this box cannot match the **Regular Expression**! Otherwise, you'll watch iTerm send text forever. Note also that this is presented for readability. For the actual value, convert to a single line with semicolons after each newline below. Finally, add `\n` to the end of the whole string.

```bash
if [[ ! $PS1 ]]; then
    echo "ERROR: Not Bash"
    sleep 60
fi
PS1='➕[\u@\h:\W]\$ '
history -c
__usher_dump() {
    echo "# com.matlowery.usher.begin"
    sleep 1
    echo "# begin ${USER-$(whoami)}@${HOSTNAME-$(hostname -f)}"
    history
    echo "# end ${USER-$(whoami)}@${HOSTNAME-$(hostname -f)}"
    echo "# com.matlowery.usher.end"
}
trap __usher_dump EXIT
```

Some notes on the above:
* Changing the prompt is critical to this solution since not changing it would simply give iTerm another regular expression match.
* `history -c` clears the history for this session (the history file is untouched).
* A function is defined and bound to the `EXIT` trap. This function dumps the history after emitting a string that another iTerm trigger can watch for. Finally, it emits a string that signals to the coprocess that the output is done.
* As a bonus, the current user and host are emitted as well. This is optional.

6. Check **Instant**.
7. Click **+** to add another trigger.
8. The **Regular Expression** must match the begin header. Here's an example:

```regexp
^# com\.matlowery\.usher\.begin$
```

9. For **Action**, choose **Run Silent Coprocess**.
10. For **Parameters**, below is a example Bash script. Save it. Make it executable. And enter the path to it in this box.

```bash
#!/usr/bin/env bash

set -euo pipefail

while read line; do
    if [[ $line =~ ^#[[:space:]]com.matlowery.usher.end ]]; then
        echo
        exit 0
    fi
    echo "$line" >> ~/.usher
done
```

11. You can leave **Instant** unchecked.

Some notes on the above:
* This shell script reads each line on stdin until it reaches the end footer.
* It appends each line to a file.

And that's it. Login to a few hosts, run some commands, and check the contents of this file.
