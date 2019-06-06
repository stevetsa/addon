## Appendix C - Proof-of-concept for running BLAST Singularity Image

[Singularity](https://singularity.lbl.gov/docs-installation) is an alternative to Docker.  

### System Configurations
Region us-east4 (Northern Virginia)
Zone us-east4c
16vCPU 104 GB Memory n1-highmem-16 
New 200 GB standard persistent disk
Ubuntu 18.04 LTS

### GCP VM Cost (June 2019)
$559.96 monthly estimate
That's about $0.767 hourly
Pay for what you use: No upfront costs and per second billing
Item    Estimated costs
16 vCPUs + 104 GB memory        $787.38/month
200 GB standard persistent disk $8.80/month
Sustained use discount  - $236.21/month
Total   $559.96/month
   
### Running BLAST Analysis
```
## Section 1. Install Singularity June 2019
## https://www.sylabs.io/guides/3.0/user-guide/installation.html

### Install dependencies
sudo apt-get update && sudo apt-get install -y \
    build-essential \
    libssl-dev \
    uuid-dev \
    libgpgme11-dev \
    squashfs-tools \
    libseccomp-dev \
    pkg-config

### Install GO
export VERSION=1.11 OS=linux ARCH=amd64 && \
    wget https://dl.google.com/go/go$VERSION.$OS-$ARCH.tar.gz && \
    sudo tar -C /usr/local -xzvf go$VERSION.$OS-$ARCH.tar.gz && \
    rm go$VERSION.$OS-$ARCH.tar.gz

echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
    echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc && \
    source ~/.bashrc

go get -u github.com/golang/dep/cmd/dep
go get -d github.com/sylabs/singularity
### Go will complain that there are no Go files, but it will still download the Singularity source code to the appropriate directory within the $GOPATH.

export VERSION=v3.0.3 # or another tag or branch if you like && \
    cd $GOPATH/src/github.com/sylabs/singularity && \
    git fetch && \
    git checkout $VERSION # omit this command to install the latest bleeding edge code from master

### By default Singularity will be installed in the /usr/local directory hierarchy. 
./mconfig && \
    make -C ./builddir && \
    sudo make -C ./builddir install

### Optional bash completion
. /usr/local/etc/bash_completion.d/singularity

### Confirm installation
singularity --version
### output singularity version 3.0.3

## blastsing image has edirect commands copied from /root/edirect to /blast/bin
singularity pull docker://stevetsa/blastsing
singularity run blastsing_latest.sif

### inside container # specify full paths
blastn -version
### Output 
### blastn: 2.9.0+
### Package: blast 2.9.0, build Apr  2 2019 17:23:44
efetch -version
### Output 10.4

## Section 2. Run BLAST with Singularity using a small example

cd ; mkdir blastdb queries fasta results blastdb_custom
efetch -db protein -format fasta -id P01349 > ~/queries/P01349.fsa
efetch -db protein -format fasta -id Q90523,P80049,P83981,P83982,P83983,P83977,P83984,P83985,P27950 > ~/fasta/nurse-shark-proteins.fsa
cd blastdb_custom
makeblastdb -in ~/fasta/nurse-shark-proteins.fsa -dbtype prot -parse_seqids -out nurse-shark-proteins -title "Nurse shark proteins" -taxid 7801 -blastdb_version 5
cd ~
blastp -query queries/P01349.fsa -db blastdb_custom/nurse-shark-proteins

## output on screen

## Exit container
exit

```

```
## Section 3. RUN BLAST with Singularity at production scale
## Install Singularity if not already done
## This section assumes using recommended hardware requirements below
## 16 CPUs, 104 GB memory and 200 GB persistent hard disk

## Modify the number of CPUs (-num_threads) in Step 3 if another type of VM is used.

## Step 1. Prepare for analysis
## Create directories
cd ; mkdir -p blastdb queries fasta results blastdb_custom

## Import and process input sequences
sudo apt install unzip
wget https://ndownloader.figshare.com/articles/6865397?private_link=729b346eda670e9daba4 -O fa.zip
unzip fa.zip -d fa

### Create three input query files
### All 28 samples
cat fa/*.fa > query.fa

### Sample 1
cat fa/'Sample_1 (paired) trimmed (paired) assembly.fa' > query1.fa

### Sample 1 to Sample 5
cat fa/'Sample_1 (paired) trimmed (paired) assembly.fa' \
    fa/'Sample_2 (paired) trimmed (paired) assembly.fa' \
    fa/'Sample_3 (paired) trimmed (paired) assembly.fa' \
    fa/'Sample_4 (paired) trimmed (paired) assembly.fa' \
    fa/'Sample_5 (paired) trimmed (paired) assembly.fa' > query5.fa

### Start singularity container
singularity run blastsing_latest.sif

### Inside container
    
### Copy query sequences to $HOME/queries folder
cp query* $HOME/queries/.

## Step 2. Display BLAST databases on the GCP
update_blastdb.pl --showall pretty --source gcp

## Download nt_v5 (nucleotide collection version 5) database
## This step takes approximately 7 min.  The following command runs in the background.
cd blastdb
update_blastdb.pl --source gcp nt_v5 &
cd ..

## At this point, confirm query/database have been properly provisioned before proceeding

## Check the size of the directory containing the BLAST database
## nt_v5 should be around 68 GB
du -sk $HOME/blastdb

## Check for queries, there should be three files - query.fa, query1.fa and query5.fa
ls -al $HOME/queries

## From this point forward, it may be easier if you run these steps in a script. 
## Simply copy and paste all the commands below into a file named script.sh
## Then run the script in the background `nohup bash script.sh > script.out &`

## Step 3. Run BLAST
## Run BLAST using query1.fa (Sample 1) 
## This command will take approximately 8 minutes to complete.
## Expected output size: 3.1 GB  
blastn -query $HOME/queries/query1.fa -db $HOME/blastdb/nt_v5 -num_threads 16 -out $HOME/results/blastn.query1.denovo16s.out

## Run BLAST using query5.fa (Samples 1-5) 
## This command will take approximately 27  minutes to complete.
## Expected output size: 10.4 GB  
blastn -query $HOME/queries/query5.fa -db $HOME/blastdb/nt_v5 -num_threads 16 -out $HOME/results/blastn.query5.denovo16s.out

## Run BLAST using query.fa (All 28 samples) 
## This command will take approximately  minutes to complete.
## Expected output size: 47.8 GB  
blastn -query $HOME/queries/query.fa -db $HOME/blastdb/nt_v5 -num_threads 16 -out $HOME/results/blastn.query.denovo16s.out

## Stdout and stderr will be in script.out
## BLAST output will be in $HOME/results

## Exit Singularity container
exit

```



```

## Appendix B - Proof-of-concept for a BLAST Jupyter Notebook
*Note this is not using the official BLAST+ Docker image.

In this section, it is assumed that you have access to a VM.  

The Jupyter Notebook is a great way to combine free text description with code in the same space and the notebook can be easily shared and reproduced.

This section has been tested using GCP's free tier VM Machine type
f1-micro (1 vCPU, 0.6 GB memory)
Start the VM and connect using keypair.  

From the terminal window,

```
ssh -i <your private key> <user>@<External IP> -L 8888:0.0.0.0:8888

# Once in VM, clone the notebook-containing repository and 
# run the Docker image with the ports open 

git clone https://github.com/stevetsa/jupyter-blast-docker.git
mkdir jupyter-blast-docker/BLAST
chmod 777 jupyter-blast-docker/BLAST
cd jupyter-blast-docker
docker run -it --rm -v `pwd`:`pwd` -w `pwd` -p 8888:8888 stevetsa/jupyter-blast-docker

# Follow on screen instructions and copy-and-paste the url with token in a brower's url field.
# Then modify the URL so it has the form below # http://127.0.0.1:8888/?token=xxxxxxxxx
# Once in Jupyter Lab environment, click on the notebook "*notebook.ipynb."
# When you are done, close the notebook window and 
# press control + C in the terminal to stop the Jupyter server.

```
Stop the VM instance.  

As an alternative, you can also run this notebook using [MyBinder.org](https://mybinder.readthedocs.io/en/latest/). You will be running the notebook in MyBinder's compute resources and you do not need a GCP account or access.  
*Currently BLAST version = 2.7.1*

To start using BLAST+ in a Jupyter Notebook, click [here](https://github.com/stevetsa/jupyter-blast-docker-binder) and click on the "launch binder" button.  

