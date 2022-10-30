---
title: "💪 Power of Bash"
date: 2017-02-27T20:11:06+02:00
draft: false
---

We all tend to do our work faster because time costs money. I realized that we lose a lot of time waiting for IDE to start. I use Netbeans IDE and all our work is related to it. For Java modules, we use Maven and, of course, we have some custom goals. Some goals are pure Java, though we have some goals that use SVN for committing, logging, reverting and so on. And some of them are related to our RDBMS as we load a DB.

But the main problem is that we can do all of this only when Netbeans IDE finishes its class path scanning and other indexing background processes. This, depending on your hardware, could sometimes take up to 3-5 minutes. I didn’t like it, so I decided to make this process a bit faster.

The following main features we use, which I could improve:
* Load database
* SonarQube scanning
* Clean and build
* Pull the source code aka Git but this is for SVN
* Push the source code aka Git but this is for SVN
* SVN reset
* SVN log
* Java Deploy
![Netbeans](images/netbeans.png)

All these features are related to Maven, therefore, we can write Maven commands. Here is where the BASH comes in! We all know that bash is a very popular and flexible scripting language used by many people around the world. I decided to install and use it instead of the default CMD from Windows. However, it is really hard to remember and write a command like:
```bash
mvn clean install myPlugin:loaddb myPlugin:execmodule myPlugin:migratedb myPlugin:activemq -Pdevelopment -Dactivemq.purgeAllQueues=true
```
I could have used some plugins from GitHub that provide some features, like fast running a command. But, since I am not a terminal at this moment person, I decided to write my own bash scripts so I could get a taste of it. This would introduce me to the bash world and give me a brief view of its commands. As starting point I installed Git that provides, by default, the Bash terminal and its environment.

## Bash aliases
Bash terminal can be configured to load a file at startup. This file is called `.bash_profile`. You can create it in the `%USER_PROFILE%` directory. In this file, you can write your own bash scripts and they will be available in the terminal at runtime. I added some aliases that are bound to some functions I could run. Example:

```bash
alias r='r_registerCurrentDirectoryWithAnAllias'
alias j='j_jumpToADirectory'
alias m="maven_command"
```
Each alias value is a Bash function that is responsible for doing something. Taking in consideration that all commands are Maven based, I defined three aliases I would need:

* `M` – Run a maven command based on a pom.xml file
* `R` – Stands for **register**. I use it to add a directory with a directory alias, list them or remove one
* `J` – Jump to a directory using a directory alias

## Core and customer structure
Before we go through the implementation, I have to explain why I defined these specific aliases. Internally, we build a product for clients that hold warehouses. More or less, they all have mostly the same functionality, so we have divided our product in 2 parts:

1. The Core (common functionality)
2. The Client (customer specific functionality)

Because of this, we have different versions for these components. The Core is based on branches and tags like `1.0-SNAPSHOT` or `1.0.1`.

The Client component is based on environments:
1. `Development-SNAPSHOT`, where it may have a dependency on `1.0-SNAPSHOT`
2. `Acceptance-SNAPSHOT`
3. `Production-SNAPSHOT`

This structure creates some patterns for accessing specific project folder, like:

```
D\:ProjectA\branches\1.0-SNAPSHOT
D\:ProjectB\tags\1.0.2
D\:ProjectC\environment\development
D\:ProjectC\environment\acceptance
D\:ProjectC\environment\production
```

## Commands
### R – Register

Now let’s see what was optimized by using Bash. Using the above mentioned aliases, I can run the commands. As I mentioned `r` stands for **register**, which holds different aliases for different paths. Using `r` I can run:

* `a` – add alias;
* `ls` – list aliases;
* `r` – remove an alias.

Main function:
```bash
function r_registerCurrentDirectoryWithAnAllias()
 {
  command=$1
  alias=$2
  r_processCommand $command $alias
 }
```
Delegated function:
```bash
function r_processCommand()
{
command=$1
alias=$2
validateParameter $command
local isCommandValid=$?
 
if [ $isCommandValid == 0 ] then
case $command in "a" )
    validateParameter $alias
    local isAlliasValid=$?
    if [ $isAlliasValid == 0 ]
    then
        add $alias
    else
        echo "Missing alias"
    fi
    ;;
"ls" ) 
    initFile out_pathDir
    pathDir=$out_pathDir
    fileLines $pathDir 0
    ;;
"r" ) 
    validateParameter $alias
    local isAlliasValid=$?
    if [ $isAlliasValid == 0 ]
    then
        remove $alias
    else
        echo "Missing alias"
    fi
    ;;
esac
    else
        echo "Missing command"
fi
}
```
  Now I go to the pom.xml file of the project and run: `r a test` – which means - **register** add for the current path the alias `test`
![terminal-1](images/a1.png#center)
```bash
function add()
{
initFile out_pathDir
pathDir=$out_pathDir
 
alias=$1
currentDir=$(pwd)
 
fileLines $pathDir 1
file_lines_array_size=${#fileLines_lines[@]} 
 
if [ $file_lines_array_size == 0 ]
then
    echo "$alias=$currentDir" &gt;&gt; $pathDir 
else
    array_of_lines=${fileLines_lines[@]}
    for line in $array_of_lines
    do
        lineKeyValue $line out_key out_value
        if [ $out_value == $currentDir ]
        then
            echo "Such directory already exists on $line"
            break
        else
            echo "$alias=$currentDir" &gt;&gt; $pathDir
            break
        fi
done
fi
}
```
This command creates in background a file `.pathDir` which will be used as a storage for all the aliases. Now I have an alias inside it.\
In order to see what other aliases I have, I can write: `r ls` – this command will display all the aliases I have in the `.pathDir`

![terminal-2](images/s2.png#center)

And of course, I can remove an alias by writing: `r r test` – which means: **register** remove the alias `test` linked to any path. Now I can use `test` for any other path from the system.

![terminal-3](images/s3.png#center)
```bash
function remove()
{
initFile out_pathDir
pathDir=$out_pathDir
 
alias=$1
currentDir=$(pwd)
sed "/$alias=/d" $pathDir >> "$pathDir tmp" && mv "$pathDir tmp" $pathDir
}
```
### J – Jump

After we added an alias for a path, we can now directly jump to that directory by typing: `j iut` – it’s a simple `cd` to the referenced path.

![terminal-4](images/s4.png#center)

Main function:
```bash
function j_jumpToADirectory
 {
  alias=$1
  branch=$2
  j_jump $alias $branch
 }
```
However, this does not bring you a lot of magic since we might have tags, branches and environments for a project. Well, jump supports additional parameters to the command, like: `j iut 1.0.2` – this way we can jump specifically inside the needed folder. But in this case `iut` alias should be linked to ‘D:\ProjectA\’ , because this will be the root folder of the project. We can also jump to a tag or an environment if it exists, by writing the following command: `j iut 1.0.2` or `j iut d` where `d` stands for development environment.

![terminal-5](images/s5.png#center)

```bash
function j_jump
{
local alias=$1
local branch=$2
validateParameter $alias
local isAliasValid=$?
if [ $isAliasValid == 0 ]
then
initFile out_pathDir
pathDir=$out_pathDir
fileLines $pathDir 1
file_lines_array_size=${#fileLines_lines[@]}
 
if [ $file_lines_array_size != 0 ]
then
array_of_lines=${fileLines_lines[@]}
local isFound=1 
for line in $array_of_lines
do
lineKeyValue $line out_key out_value
if [ $out_key == $alias ]
then
isFound=0
local path=$out_value
validateParameter $branch
local isBranchValid=$?
if [ $isBranchValid != 0 ]
then
cd $path
break
else
cd $path
j_process_branch $branch out_path
local fullpath="$path/$out_path"
cd $fullpath
break
fi
fi
done
if [ $isFound == 1 ]
then
echo "No such alias found"
fi
else
echo "No aliases found"
fi
else
echo "Missing parameter"
fi
}
 
function j_process_branch
{
local branch=$1
local pth=$2
isMajorMinorFunc $branch
local isMajorMinor=$?
isTagFunc $branch
local isTag=$?
isPrivateNumberFunc $branch
local isPrivateBranch=$?
 
if [ $isMajorMinor == 0 ]
then
    eval $pth="branches/$branch-SNAPSHOT"
elif [ $isTag == 0 ]
then
    eval $pth="tags/$branch"
elif [ $isPrivateBranch == 0 ]
then
    eval $pth="branches//private-$branch"
else
case $branch in
    "t" )
    eval $pth="branches/TRUNK-SNAPSHOT"
    ;;
"d" )
    eval $pth="environment/development"
    ;;
"a" )
    eval $pth="environment/acceptance"
    ;;
"p" )
    eval $pth="environment/production"
    ;;
esac
fi
}
```
### M – Maven

Now, when we are able to jump faster from one project to another, we can run maven goals over the pom.xml files, like this:

* `m ldb` – loads database
* `m pull` – SVN update goal
* `m push` – SVN commit goal
* `m reset` – SVN revert goal
* `m log` – SVN log goal
* `m deploy` – deploy
* `m sonar` – sonar goal

![terminal-6](images/s6.png#center)

```bash
function maven_command
{
local command=$1
validateParameter $command
local isCommandValid=$?
if [ $isCommandValid == 0 ]
then
case $command in
    "sonar" )
    mvn clean install sonar:sonar -Dsonar.host.url="https://sonarqube-host:9000"
    ;;
"ldb" )
    local setup=$(find -regex ".*setup$" | head -1)
    cd "$setup"
    mvn clean install myPlugin:loaddb myPlugin:execmodule myPlugin:migratedb myPlugin:activemq -Pdevelopment -Dactivemq.purgeAllQueues=true
    ;;
"ci" )
    mvn clean install
    ;;
"pull" )
    mvn myPlugin:ts-update -N
    ;;
"push" )
    mvn myPlugin:ts-commit -N
    ;;
"reset" )
    mvn myPlugin:ts-revert -N
    ;;
"log" )
    mvn myPlugin:ts-log -N
    ;;
"deploy" )
    mvn clean source:jar javadoc:jar deploy -Pjavadocprof
    ;;
 
*)
    mvn $@
    ;;
esac
else
    echo "Missing command"
fi
}
```

## Conclusion

Having all of this configured, we can run the load the database or any command over a `pom.xml` file without even having to start the Netbeans IDE. The `.bash_profile` can be customized per developer, so everyone can add commands of their own. The most important thing is that using Bash we can gain a lot of power due to its internal commands that Windows does not have. We save a lot of time. And while Netbeans starts, you could already have 3-4 terminals working:

* One loading the DB
* Another loading logs
* Third updating another project