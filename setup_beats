#!/bin/bash -x
# This script is used to install and configure Beats (filebeat, topbeat, packetbeat)
cd /tmp
source /etc/secops/sensor.conf
if [ $? > 0 ]; then
  if [ -e /etc/filebeat ]; then $FILEBEAT=no; fi
  if [ -e /etc/topbeat ]; then $TOPBEAT=no; fi
  if [ -e /etc/packetbeat ]; then $PACKETBEAT=no; fi
fi

# Set default cluster for monitoring
PROD_OR_DEV="DEV"

# Get user input (to be added later)
# read -e -i "$PROD_OR_DEV" -p "Which ES Cluster would you like to send Beats data to (PROD/DEV)? Enter for Default: " input
#PROD_OR_DEV="${input:-$PROD_OR_DEV}"

# Determine which cluster to send Alerts to
echo "PROD_OR_DEV is set to $PROD_OR_DEV"
shopt -s nocasematch
if [ "$PROD_OR_DEV" == "DEV" ]; then
    ELASTIC_IP="10.233.10.121"
elif [ "$PROD_OR_DEV" == "PROD" ]; then
    ELASTIC_IP="10.233.10.101"
else
    echo "PROD_OR_DEV must be set to DEV or PROD"
    shopt -u nocasematch
	exit 1
fi
shopt -u nocasematch

# Determine Linux Distribution and Architecture

source /etc/os-release

ARCH="$(uname -mrs | cut -d' ' -f3)"
echo "You appear to be running $PRETTY_NAME on $ARCH Architecture"

if [ "$ARCH" == "x86_64" ]; then
    BUILD="amd64"
elif [ "$ARCH" == "i686" ]; then
    BUILD="i386"
else
    echo "Unable to determine system architecture"
    exit 1
fi

if [[ $ID =~ (ubuntu|debian) ]]; then
    apt-get -y install curl
else
    echo "This installer does not support your Linux Distro at this time"
	exit 1
fi

beats () {
####### Setup filebeat ##########
yesno="y"
if [[ $FILEBEAT == "yes" ]]; then
    echo "Filebeat is already installed."
	while [[ ! $yesno =~ (y|n) ]]; do
	  read -e -i "n" -p "Would you like to proceed anyway? (y/n) : " yesno
    done
fi
### Download and install filebeat
if [ ! $yesno == "n" ]; then
    curl -L -O https://download.elastic.co/beats/filebeat/filebeat_1.0.1_$BUILD.deb
    dpkg -i filebeat_1.0.1_$BUILD.deb
	### configure settings
    mv /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak
    ########## START ##############
    #   copy/paste filebeat.yml   #
    ########## END  ###############

    ### install GeoLiteCity.dat
    curl -L -O http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
    gzip -d GeoLiteCity.dat.gz
    mkdir -p /usr/share/GeoIP/
    mv /tmp/GeoLiteCity.dat /usr/share/GeoIP/

    ### Upload elasticsearch mapping
    curl -XPUT "http://$ELASTIC_IP:9200/_template/filebeat?pretty" -d@/etc/filebeat/filebeat.template.json

else
    echo "Skipping Filebeat Setup"
fi

######## Setup topbeat ##########
if [[ $TOPBEAT == "yes" ]]; then
    echo "Topbeat is already installed."
	yesno=""
	while [[ ! $yesno =~ (y|n) ]]; do
	  read -e -i "n" -p "Would you like to proceed anyway? (y/n) : " yesno
    done
fi
if [ ! $yesno == "n" ]; then
    ### Download and run setup
    curl -L -O https://download.elastic.co/beats/topbeat/topbeat_1.0.1_$BUILD.deb
    dpkg -i topbeat_1.0.1_$BUILD.deb

    ### configure settings
    mv /etc/topbeat/topbeat.yml /etc/topbeat/topbeat.yml.bak

    ########## START ##############
    #   copy/paste topbeat.yml    #
    ########## END  ###############

    ### Upload elasticsearch mapping
    curl -XPUT "http://$ELASTIC_IP:9200/_template/topbeat" -d@/etc/topbeat/topbeat.template.json
else
    echo "Skipping Topbeat Setup"
fi

######## Setup packetbeat ##########
if [[ $PACKETBEAT == "yes" ]]; then
    echo "Packetbeat is already installed."
	yesno=""
	while [[ ! $yesno =~ (y|n) ]]; do
	  read -e -i "n" -p "Would you like to proceed anyway? (y/n) : " yesno
    done
fi
if [ ! $yesno == "n" ]; then
    ### Download and run setup
    sudo apt-get install libpcap0.8
    curl -L -O https://download.elastic.co/beats/packetbeat/packetbeat_1.0.1_$BUILD.deb
    dpkg -i packetbeat_1.0.1_$BUILD.deb

    ### configure settings
    mv /etc/packetbeat/packetbeat.yml /etc/packetbeat/packetbeat.yml.bak

    ########## START ##############
    #  copy/paste packetbeat.yml  #
    ########## END  ###############

    ### Upload elasticsearch mapping
    curl -XPUT "http://$ELASTIC_IP:9200/_template/packetbeat" -d@/etc/packetbeat/packetbeat.template.json
else
    echo "Skipping Packetbeat Setup"
fi
}

dashboards () {
### Load sample dashboards in kibana for beats
curl -L -O http://download.elastic.co/beats/dashboards/beats-dashboards-1.0.1.tar.gz
tar xzvf beats-dashboards-1.0.1.tar.gz
cd beats-dashboards-1.0.1/
./load.sh "http://$ELASTIC_IP:9200/"
cd /tmp
}

start () {
### start and verify presence in Elasticsearch

update-rc.d filebeat defaults 95 10
update-rc.d topbeat defaults 95 10
update-rc.d packetbeat defaults 95 10

cat > /usr/local/bin/restart_beats.sh <<EOF
service filebeat restart
service topbeat restart
service packetbeat restart
EOF
chmod +x /usr/local/bin/restart_beats.sh
/usr/local/bin/restart_beats.sh
}

verify () {
sleep 30
curl -XGET "http://$ELASTIC_IP:9200/filebeat-*/_search?pretty"
curl -XGET "http://$ELASTIC_IP:9200/topbeat-*/_search?pretty"
curl -XGET "http://$ELASTIC_IP:9200/packetbeat-*/_search?pretty"
}

cluster () {
shopt -s nocasematch
if [ "$PROD_OR_DEV" == "DEV" ]; then
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.121:9200\",\"10.233.10.123:9200\",\"10.233.10.123:9200\"\]/g' /etc/filebeat/filebeat.yml
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.121:9200\",\"10.233.10.123:9200\",\"10.233.10.123:9200\"\]/g' /etc/topbeat/topbeat.yml 
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.121:9200\",\"10.233.10.123:9200\",\"10.233.10.123:9200\"\]/g' /etc/packetbeat/packetbeat.yml 
    shopt -u nocasematch
elif [ "$PROD_OR_DEV" == "PROD" ]; then
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.101:9200\",\"10.233.10.103:9200\",\"10.233.10.103:9200\"\]/g' /etc/packetbeat/filebeat.yml 
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.101:9200\",\"10.233.10.103:9200\",\"10.233.10.103:9200\"\]/g' /etc/topbeat/topbeat.yml 
    sed -i 's/hosts: \[placeholder\]/hosts: \[\"10.233.10.101:9200\",\"10.233.10.103:9200\",\"10.233.10.103:9200\"\]/g' /etc/packetbeat/packetbeat.yml 
shopt -u nocasematch
else
    echo "PROD_OR_DEV must be set to DEV or PROD"
    shopt -u nocasematch
	exit 1
fi
}

filebeat () {
curl -L -O https://raw.githubusercontent.com/rlwmmw/mybeats/master/filebeat.yml 
mv /tmp/filebeat.yml /etc/filebeat/filebeat.yml
#echo "# copy/paste the contents of filebeat.yml below this line" >/etc/filebeat/filebeat.yml
#vim /etc/filebeat/filebeat.yml
}

topbeat () {
curl -L -O https://raw.githubusercontent.com/rlwmmw/mybeats/master/topbeat.yml 
mv /tmp/topbeat.yml /etc/topbeat/topbeat.yml
#echo "# copy/paste the contents of topbeat.yml below this line" >/etc/topbeat/topbeat.yml
#vim /etc/topbeat/topbeat.yml
}

packetbeat () {
curl -L -O https://raw.githubusercontent.com/rlwmmw/mybeats/master/packetbeat.yml
mv /tmp/packetbeat.yml /etc/packetbeat/packetbeat.yml
#echo "# copy/paste the contents of packetbeat.yml below this line" >/etc/packetbeat/packetbeat.yml
#vim /etc/packetbeat/packetbeat.yml
}

beats;
filebeat;
topbeat;
packetbeat;
cluster;
#dashboards;
start;
# verify;
