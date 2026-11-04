# Ralph Loops

<img src=img/pooh.png width=300px align=right />

The Ralph Loop is one of the most common AI agents.
This lab has you write your own Ralph loop and use it to solve a multi-step problem in a repo.
At the end, an optional final step walks you through using the Ralph loop to solve the last pagila homework.

We also introduce the idea of a *git worktree*.
Worktrees are commonly used in AI workflows to allow "swarms" of AI agents to work in parallel on different tasks in the same project.

## Part 0: Project Setup

The markdown-compiler project is one of the first assignments from the intro to computer science course at CMC (CSCI040).
The project takes arbitrary markdown code as input and converts it to HTML.
For example, it converts the string `*example*` to `<i>example</i>`.

In this lab, we will explore good workflows for getting AI to implement the project for us automatically.

First, lets clone a fork of the markdown-compiler assignment.
```
$ git clone https://github.com/mikeizbicki/markdown-compiler-solutionish
$ cd markdown-compiler-solutionish
```
The code is in the `markdown_compiler` folder and the instructions for the assignment in the `example/README.md` file.
The final goal of the assignment is to convert this `README.md` file into an html file.
Because the code is mostly completed, you can do that with the following commands:

1. First install the project "locally" with the command.
    ```
    $ pip3 install -e .
    ```
    The `.` refers to the current folder.
    Recall that python projects are defined by the `pyproject.toml` file,
    and so `pip3` will find the `./pyproject.toml` file and install the project based on its contents.

    The `-e` stands for "edit",
    and is commonly used when installing a project from the harddrive.
    By default when you run `pip3 install .` the python files in the current folder are *copied* to the venv that you are installing into;
    this means that as you update files in the current folder,
    the installed files do not get updated,
    and everytime you make an edit you need to re-run pip install.
    Adding the `-e` flag causes `pip3` to create a *symlink* instead of a copy,
    and so the files in your venv are always automatically kept up-to-date,
    and no reinstall is needed as you edit your python files.

2. Run the actual compiler on the instructions README:
    ```
    $ markdown-compiler --input_file=example/README.md
    ```
    This should create a file `example/README.html`,
    and you should briefly verify it contains HTML.
    ```
    $ head example/README.html
    ```
    And you can view that html in a terminal web browser
    ```
    $ links example/README.html
    ```

## Part 1: git worktrees

Even though the project seems to work, it's actually broken.
Run the test script and observe that it fails
```
$ ./test.sh
```

> **ASIDE:**
> AI is exceptionally good at writing production quality code when it is provided with good test cases
> (and it has been since at least 2023).
> Many experienced developers use a *test driven development* (TDD) workflow with AI where the developer writes the test cases and then tells AI to implement the functions.
> This is a good way for the developer to still stay in touch with the code and "drive" the model effectively.
> A major growth area for most students / junior developers is that you all refuse to write tests for your code.
>
> <img src=img/tdd.jpg width=250px />
>
> A lack of tests has always made writing code painful, but it makes writing code with AI much more painful.

Most git tutorials would suggest creating a new *branch* at this point.
But we will create a *worktree* instead.
A worktree is a git branch and a new directory where the branch's contents are stored at.
The advantage of a worktree over a branch is that we can have two copies of the repo's source checked out simultaneously.
Git worktrees were originally invented for standard non-AI workflows (they landed in git in 2015),
but they have become a key component in letting AI agents work autonomously in parallel.

Create a new worktree by running
```
$ git worktree add ~/markdown-bugfix
```
Observe that this created a new branch in your repo but you still have `master` checked out.
```
$ git branch
  markdown-bugfix
* master
```
The `git worktree add` command takes a *path* as an argument,
and a copy of the current repo will be placed at that location.
(The name of the branch it creates is the name of the last folder in the path.)
In our case, we specified that the worktree should be located at `~/markdown-bugfix`, so lets cd there and verify it has the same contents as out current folder:
```
$ ls
example  markdown_compiler  pyproject.toml  README.md  requirements.txt  test.sh
$ cd ~/markdown-bugfix
$ ls
example  markdown_compiler  pyproject.toml  README.md  requirements.txt  test.sh
```
but now inside this folder we are in the `markdown-bugfix` branch.
```
$ git branch
* markdown-bugfix
  master
```

At this point, we are ready to write our code to fix the bug.

The following command should result in a fixed test case:
```
$ committe -m openrouter+deepseek <<EOF
$(files-to-prompt .)

$(./test.sh)

fix the broken test
EOF
```
<!--
Instructor Note:
The test cases are stored in a hidden folder so that files-to-prompt does not catch them.
This fact combined with the fact that ./test.sh internally uses the pytest -x command hides all the context from a model needed to 1-shot the solution.
-->

Recall that the most important skill for using AI effectively is writing good prompts.
Above, I am using command substitution to ensure that all of the source code in the repo and the output of the test cases are both included in the prompt.

A less important skill than controlling the prompt is deciding which model to use.
The deepseek model is a good general-purpose model that is very cheap and essentially SOTA at coding tasks.
This particular task is easy to do and could be solved with a much older model,
but deepseek is cheap enough I basically use it for everything,
and the prompt above is too large for the free tier that groq provides.

Verify the code written by checking the diff:
```
$ git diff HEAD~1
```
You should see a bunch of correct looking code added to the `compile_headers` function.

> **RECALL:**
> `HEAD` refers to the current commit checked out,
> and `~N` refers to the Nth parent.
> So `HEAD~1` combines to give us the parent of our current commit.
> The incantation above is a common trick for reviewing recent changes to code and worth memorizing.

Now lets re-run the tests:
```
$ ./test.sh
```

Oh-no!
We're still failing!

Why?!

If you look closely at the pytest output, you should observe that we are passing the original test case.
But now we are failing a new, different test case.

You should `cd` between your two worktrees at this point and verify that they are failing for different reasons.

So how should we fix this new test failure?

Lets just repeat the procedure above until all tests pass.

## Ralph Loop

Repeatedly calling an LLM with the same prompt over and over again is called a Ralph loop.
It is the stupidest possible coding agent and one of the most effective.

The code below creates a new shell script in your home folder called `ralph.sh` which is the loop.

```bash
$ cat > ~/ralph.sh <<'EOFF'
#!/bin/bash

# In order for the script to have access to the `committe` function,
# we need to initialize dic here.
# This is because bash functions happen at the process level,
# and a shell script is a different process than your interactive bash repl.
eval "$(dic --init)" 2>/dev/null

# the loop is below
for i in $(seq 0 99); do
    echo '----------------------------------------'
    echo "ralph loop i=$i"
    echo '----------------------------------------'
    if sandbox ./test.sh > /tmp/out 2>&1; then
        echo "tests pass after $i rounds"
        exit 0
    fi
    committe "$@" <<EOF
$(files-to-prompt markdown_compiler/)

$(cat /tmp/out)

fix the broken test
EOF
done
EOFF
```
In the code above, observe that:
1. We use nested heredocs.
    The inner-most heredoc inside the shell script uses `EOF` as the file terminator and the outermost heredoc uses `'EOFF'`.
    The quoting of both of these heredocs is critical for the variables to be interpreted correctly.
2. The test cases are wrapped inside of `sandbox`.
    The `./test.sh` script will be running code that was written by an AI and that we have not reviewed... this code could do anything... but `sandbox` will ensure that it can't break our computer or commit any felonies.
3. `committe` is called with the `"$@"` parameter.
    Recall that `"$@"` is a special parameter in bash scripts that refers to the parameters that were passed into the script.
    Thus, doing something like `ralph.sh -m openrouter+deepseek` will pass the `-m openrouter+deepseek` parameters on to `committe`, which will pass them on to `dic`.
    We can also control our prompt history by passing in the `-c` or `--mid=` parameters.

Make it executable with the command
```
$ chmod a+x ~/ralph.sh
```

And now let the llm go brr....

<img src="img/llm-maximalism_meme.jpg" width=300px />

```
$ cd ~/markdown-bugfix # ensure we're in the right worktree
$ ~/ralph.sh -m openrouter+deepseek
```
My code took about 5 minutes and $0.03 to finish.
But since everything about llms is nondeterministic, YMMV.

> **NOTE:**
> If everything is so deterministic, how am I so confident that the LLMs won't be "1-shotting" the problem and that we'll actually need to loop?
> Two pieces of careful prompt engineering.
> 1. I put all the test cases in a hidden folder `.github/tests`.
>   Because `files-to-prompt` ignores hidden files, this ensures that the LLM will not actually be able to see the full test suite.
> 2. The `test.sh` script uses `pytest -x`.
>   The `-x` flag means that pytest will stop immediately on the first failing test and not run the full test suite.
>   (This is commonly used for test suites that take a long time to run.
>   Many production test suites take hours to run and cost $$$, and so we put the basic quick tests first so that if they fail we don't have to waste our time and money.)
>
> Notice that I did not ask the model to only run one test... models are evil and will not obey your instructions... I made it impossible for the model to misbehave and run more than one test at a time.
> That is the essence of good prompt engineering... making it impossible for the model to misbehave.

Once your ralph loop above finishes, take a look at the git logs
```
$ git log --oneline master..HEAD
```
Observe that the loop has made many commits.
(The `master..HEAD` syntax shows us all of the commits between the `master` branch and our current commit.)

This is ugly.

In general, we only want to have commits in our git history that we might actually want to checkout someday.
And we know that all of these intermediate commits are broken,
and so we will never want to checkout any of them.
So it would be good practice to *squash* these commits into a single commit before merging.

This is easy to do with the following two commands:
```
$ git reset --soft master
$ git commit -m 'fix the markdown compiler'
```
> **NOTE:**
> The `git reset --soft` command above is a bit subtle.
> It works by moving our *branch pointer* to `master` without changing the contents of the *working tree* (what git calls the actual files on the hard drive).
> So when we run the subsequent `git commit` command,
> we are creating a new commit based off of master with the contents of all the previous commits.
>
> Obvious, right?
>
> <img src=img/learn-git.jpg width=200px />

And now we are only a single commit ahead of `master`.
```
$ git log --oneline master..HEAD
```
At this point we are ready to merge back into `master` and delete the worktree...
but instead of doing that we will explore the `itera` script.

## itera

The geni library contains a command `itera` (latin for the imperative "iterate") that implements the ralph loop above along with a number of quality of life improvements.
For example it:
1. creates the worktree automatically
2. squashes commits automatically
3. merges the final commit automatically
4. provides a percent complete timer for `./test.sh` based on historic runtimes

Test the `itera` script by first `cd`ing back to your main repo folder and run
```
$ itera -m openrouter+deepseek <<EOF
$(files-to-prompt .)

fix the test cases
EOF
```
You'll likely see two errors when running the command above:
1. `itera` by default enforces that we have passing test cases before running.
    (This is to ensure that when we are using it to add new features we start off in a sane/working state.)
    You can disable this initial check by adding the `--allow-red` flag to the itera command.
2. `itera` internally calls `committe`, and `committe` refuses to run if the git repo is not clean.
    (This is to ensure that an LLM commit won't overwrite uncommitted human work.)
    The previous commands have created a few files in the current folder that are not tracked by git and so the repo is not clean.
    You can view what these files are with
    ```
    $ git status
    ```
    Observe they are not important, and then you can delete them with
    ```
    $ git clean -fd
    ```
    (The command above deletes all non-tracked files and is a useful git incantation to memorize.)

Applying these fixes, itera should go brr for a few minutes again and then give a nice clean commit.

<img src=img/ralphing.jpg width=300px />

## Submission

Push your `markdown-compiler-solutionish` repo to github with the fix added.

### Optional Pagila

The ralph loop with deepseek is powerful enough to complete all of the pagila programming assignments.
You just have to be a bit careful about how you prompt and how you sandbox.

For prompting, you will need to include:
1. The questions that need answering.
    This is easy to do with something like `$(files-to-prompt sql)`
    (because each file in the `sql` folder is one of the questions).

2. The database schema.
    This is harder to do because the database is too large for an LLM context window.
    The "obvious" thing to try is to just include the whole pagila repo,
    but this is over 4 million tokens:
    ```
    $ files-to-prompt pagila/ | ttok
    4297466
    ```
    (Recall that `ttok` counts the number of tokens on stdin.)

    The problem is that the repo has both data and schema inside of it,
    and we want only the schema.
    In psql the `\d+ tablename` command gives us the schema of a particular table---tablename can include glob patterns and so  `\d+ *` gives us the schema of all tables.
    Recall also that a standard convention of repl programs is that `-c` takes an argument to run inside the repl and so `psql -c '\d+ *'` is the shell command to give us the db schema.
    But we can't directly run this command from the lambda server
    (because the postgres database is inside a container),
    and so it needs to be run inside of docker.
    The final incantation that will give us the full schema inside the lambda server looks like
    ```
    docker compose exec pg psql -c '\d+ *'
    ```

3. The failing test cases.
    The current `run_tests.sh` script computes which tests fail,
    but it doesn't provide any output about why the tests fail which the LLM would need.
    Modify the script so that whenever a test fails you also print a useful failure message that the LLM can act on.
    (This could be just the diff, just the raw expected/results outputs, or both.)

For sandboxing:
1. Recall that the `itera` script runs all test cases inside of `sandbox` command.
    (This is because it is running potentially unsafe LLM code.)
    `sandbox` by default allows only read access to the current folder,
    and this breaks docker (which requires read/write access elsewhere in the filesystem).
    The easiest way for you to solve this problem is to edit your `ralph.sh` command that you wrote above to remove the `sandbox` invocation.
    This is mostly safe because the actual LLM code will be running inside a docker container, and so it can't get into too much trouble no matter what it does.

    (There are more sophisticated things that you could do for a stronger sandbox, but that's beyond the scope of what you should worry about for now.)
