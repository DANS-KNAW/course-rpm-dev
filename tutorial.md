Tutorial
========

The following exercises are intended to help you understand the make-up of a project
generated with [easy-module-archetype], in particular how it supports creating RPM 
packages. Please, do *not* skip the exercises that ask seemingly lame questions, like
"where is such-and-such configured". By verifying these things you will get a grasp of
how things work much faster.

[easy-module-archetype]: https://github.com/DANS-KNAW/easy-module-archetype


Preparation
-----------

### Software to install
Make sure you have the following installed on your Mac. (Oh, yes, the instructions assume that you
are working on a Mac. However, there is not Mac-only software used (AFAIK), so you should be able to 
do this tutorial on a different platform.) The list below was copied from [easy-dtap], so if you are
an EASY developer, you should already have all of this:

* Brew, to install some of the other stuff: see [brew](http://brew.sh) if you haven't installed it yet.
* [Java 8](http://www.oracle.com/technetwork/java/javase/downloads/index.html).
* [Vagrant 1.9.4](http://vagrantup.com).
* Ansible 2.3.0.0: `brew install ansible` (preferred) or `sudo pip install ansible==2.3.0.0`
* Maven 3.3.3 or higher: `brew install maven` 
* [VirtualBox 5.1.22](https://www.virtualbox.org/wiki/Downloads). (Downloadlink or `brew cask install virtualbox`.)
* The `vagrant-vbguest` plugin for vagrant: `vagrant plugin install vagrant-vbguest`.
* RPM: `brew install rpm`.

[easy-dtap]: https://github.com/DANS-KNAW/easy-dtap#requirements

### DANS GitHub projects to clone
Clone the following projects to you computer:

* https://github.com/DANS-KNAW/easy-module-archetype - this project contains the `generate-easy-module.sh`
  script that you will be calling.
* https://github.com/DANS-KNAW/dans-parent - you don't actually need to have it locally, but we will refer to 
  to `pom.xml` and other source files in this project in the course of the tutorial.

Put clone on your `PATH` variable, so that you can call the scripts in it from any directory.

### Hosts entry
Make a new entry in the file `/etc/hosts` on your computer:

    192.168.33.32   test.dans.knaw.nl
    
This will allow you to access the virtual machine that is created during the tutorial, by this 
hostname.


Generating a project
--------------------
We will first generate a fresh project. 

1. Open a new terminal window and go to the directory in which you want to create the new project
   as a sub-directory.
2. Type `generate-easy-module.sh` and Enter.
3. The archetype plugin now asks a couple of questions about the project to be created: 
    * easy-module-archetype version? - go with the default, that is: hit Enter.
    * Module artifactId (e.g., easy-test-module): - type `easy-tutorial`
    * Name module's main package (i.e. the one under nl.knaw.dans.easy): - type `tutorial`
    * Description (one to four sentences): - Type any description you like.
4. Hit Enter again when asked for confirmation about the parameters.

A number of things will now happen in sequence:
* The project is generated
* The `init-project.sh` script is run, which will do some things like:
    - Fill in license headers
    - Build the project (including the RPM package).
    
The project generation process is not the focus of the present tutorial, so if you want to know exactly 
what `init-project.sh` does, read the source file, which is in the root of the generated project. 

Starting and provisioning a local VM
------------------------------------

Now that we have our new project, let's try it out. 

1. Or wait, first better let us store the initial version in a git repo. `cd easy-tutorial ; git init ; git add . ; git commit -m "Initial commit"`
2. We could now start the project for debugging with one of the `run*.sh` scripts. However, we can now
   also start a test VM, install the RPM that was just built and run the service exactly as it will be
   in the eventual production environment. Let's do that:
   
        vagrant up

   This will start the VM and trigger the Ansible provisioner to do some boilerplate configuration and install the
   RPM.
   * *Question 1: what provisioner does vagrant use?* 
   * *Question 2: what file is the playbook for this provisioner?* 
     
   
3. If the previous command terminated successfully, let's see if the service is running:

        curl http://test.dans.knaw.nl
        
   This should result in the following message being returned:
            
        EASY Tutorial Service running...

   * *Question 4: on what port does the easy-tutorial daemon listen?* 
   * *Question 5: How is your request to tcp port 80 on the VM relayed to this port? 
      Where is this configured and how is this configuration applied to the VM?* 

4. SSH in on the virtual machine: `vagrant ssh`
5. List the installed RPM packages that start with "dans": `yum list installed dans*`(As you can see, you 
  can use [glob patterns] here.)	

   * *Question 6: What version of the module is installed?*
   
[glob patterns]: https://en.wikipedia.org/wiki/Glob_%28programming%29

Creating a new version
----------------------

We are going to add some functionality to the `easy-tutorial` service. The code to add will be supplied
so you can focus on understanding how the new version is packaged as an RPM, published in a YUM repo and
then installed on the VM.

The new version will accept `POST` requests from clients. The requests will contain a simple html
form with questionnaire data and the service will simple parrot back the data for now.
 
1. Add the following code snippet to `nl.knaw.dans.easy.tutorial.EasyTutorialServlet`:

       post("/answers") {
           contentType = "text/html"
           val name = params("name")
           val age = params("age")
           val language = params("language")
   
           Ok(s"""
             <html><head></head><body>
             Hi $name ($age). So, you like $language.
             </body></html>
             """)
       }

2. Recompile (in IntelliJ) and test the code with `./run-service.sh` and then filling in the form
   `questionnaire-localhost.html` in your browser and submitting it.
3. Rebuild the project with `mvn clean install`. This will also create a new RPM package.
4. Before we can install this package on the server we must publish it in the local yum repository.
   To do this run `./rebuild-repo.sh`
   
   * *Question 7: what playbook does `rebuild-repo.sh` run?* 
   * *Question 8: where is the local yum repository located? On your Mac, on the VM?*
   * *Question 9: how does yum know where the repo is?*
   
5. SSH in on your VM with `vagrant ssh`.
6. Check that yum is aware that there is an update for the module: 

       yum check-update dans*
    
7. Upgrade the module:

       sudo yum upgrade dans*

8. Start the upgraded service:

       sudo service easy-tutorial start
       
9. Now test the service on the VM using `questionnaire-testvm.html`


Zooming in
----------
To get a better grasp of what is going on, we are now going to create yet another upgrade. In the 
new version the questionnaire answers will be stored in a simle SQLite database. The upgrade package
will create the database if it does not exist yet.
 
1. We will first add the dependency to the SQLite JDBC driver to our pom:
 
       <dependency>
           <groupId>org.xerial</groupId>
           <artifactId>sqlite-jdbc</artifactId>
       </dependency>
   
2. Though strictly not necessary, we will also add `sqlite` as a required package to our RPM: in the
   `pom.xml` go to the configuration of `rpm-maven-plugin` and add `<require>sqlite</require>` after
   the `jsvc` require.
3.    
 
 
 


















References
----------
* RPM website: http://rpm.org/
* Very useful, and extensive book about RPM: http://rpm5.org/docs/max-rpm.pdf
* YUM man page: https://linux.die.net/man/8/yum
* 
