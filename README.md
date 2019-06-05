# addon

## Appendix C - Proof-of-concept for running BLAST Singularity image

[Singularity](https://singularity.lbl.gov/docs-installation) is an alternative to Docker.  
    
```
### install singularity

git clone https://github.com/singularityware/singularity.git
cd singularity
./autogen.sh
./configure --prefix=/usr/local --sysconfdir=/etc
make
sudo make install

### install dependencies
sudo apt-get update && \
    sudo apt-get install \
    python \
    dh-autoreconf \
    build-essential \
    libarchive-dev\
    docker.io

sudo apt-get install -y singularity-container

singularity pull docker://ncbi/blast
singularity run blast.simg 

## inside container # specify full paths

## issue - Singularity does not have root access - need to move edirect executables out of root/

makeblastdb -in ~/fasta/nurse-shark-proteins.fsa -dbtype prot \
-parse_seqids -out ~/blastdb/nurse-shark-proteins \
-title "Nurse shark proteins" -taxid 7801 -blastdb_version 5 

blastp -query ~/queries/P01349.fsa -db ~/blastdb/nurse-shark-proteins \
-out ~/results/blastp.out 

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

To start using BLAST+ in a Jupyter Notebook, click [here](https://github.com/stevetsa/jupyter-blast-docker-binder) and click on the "launch binder" button
