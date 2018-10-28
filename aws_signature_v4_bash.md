# AWS Authentication
# Bash script
## Cloud CLI interfaces
I have come across a problem where I had to work with a legacy system and import the data into cloud based data storage. Because of imposed constraints, the cloud I had to deal with was AWS. On the other hand, the environment I was placed in didn’t really allow for anything else but shell.

The problem at hand was to upload data exported from legacy system to cloud storage.
I have quite a bit of experience with Google Cloud Platform where this type of operation would be performed with their `gsutil` tool that maps popular shell commands like `cp` - the command there would look like this:
gsutil cp <source file location> <target file location>
You have to of course remember to setup credentials file in the appropriate place but that’s easy to sort out.

In AWS, you have to use the REST API to upload file into specified location. In theory it’s a simple operation: form and send HTTP request to an AWS service. There is one little wrinkle though: you have to take care of the authentication. They are using something called AWS Signature Version 4 (https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-authenticating-requests.html).

In the documentation, there are plenty of examples how to create this signature in JavaScript, Java, Python, etc. I couldn’t find anything that would document it in bash though. There are some scattered examples on GitHub but they often don’t seem to work out of the box so I thought I post it here for future reference and to make life easier for fellow AWS API users.
## Script outline
The script has two dependencies and 3 parts.
### Dependencies
- Openssl - for encoding content with usage of SHA256
- Curl - for assembling and sending the actual HTTP request
### Script parts
- Parameter parsing
This is a standard thing. Not much to explain here but it’s needed to make the script reusable:
```bash
	POSITIONAL=()
while [[ $# -gt 0 ]]
do
key="$1"

case $key in
-f|--file-name)
FILE_NAME="$2"
shift # past argument
shift # past value
;;
-u|--file-to-upload)
FILE_TO_UPLOAD="$2"
shift # past argument
shift # past value
;;
-a|--access-key)
ACCESS_KEY="$2"
shift # past argument
shift # past value
;;
-s|--secret-access-key)
SECRET_ACCESS_KEY="$2"
shift # past argument
shift # past value
;;
*)    # unknown option
POSITIONAL+=("$1") # save it in an array for later
shift # past argument
;;
esac
done
set -- "${POSITIONAL[@]}" # restore positional parameters
```
There are for variables set here setting the name of the file to upload, how it should be called when uploaded and login credentials (set in AWS IAM console).

- Helper functions for encrypting components of the request
```bash
sha256_hash(){
 a="$@"
 printf "$a" | openssl dgst -binary -sha256
}
sha256_hash_in_hex(){
 a="$@"
 printf "$a" | openssl dgst -binary -sha256 | od -An -vtx1 | sed 's/[ \n]//g' | sed 'N;s/\n//'
}
# adapted from: http://stackoverflow.com/questions/7285059/hmac-sha1-in-bash
# and also this answer there: http://stackoverflow.com/a/22369607
function hex_of_sha256_hmac_with_string_key_and_value {
 KEY=$1
 DATA="$2"
 shift 2
 printf "$DATA" | openssl dgst -binary -sha256 -hmac "$KEY" | od -An -vtx1 | sed 's/[ \n]//g' | sed 'N;s/\n//'
}
function hex_of_sha256_hmac_with_hex_key_and_value {
 KEY="$1"
 DATA="$2"
 shift 2
 printf "$DATA" | openssl dgst -binary -sha256 -mac HMAC -macopt "hexkey:$KEY" | od -An -vtx1 | sed 's/[ \n]//g' | sed 'N;s/\n//'
}
#####################################################################################################################

echo $FILE_TO_UPLOAD

unameOut="$(uname -s)"
case "${unameOut}" in
   Linux*)     FILE_SIZE=$(stat -c%s "$FILE_TO_UPLOAD");;
   Darwin*)    FILE_SIZE=$(stat -f%z "$FILE_TO_UPLOAD");;
   CYGWIN*)    FILE_SIZE="0";;
   MINGW*)     FILE_SIZE="0";;
   *)          FILE_SIZE="0"
esac

FILE_CONTENT="$(cat $FILE_TO_UPLOAD)"
CONTENT_HASH="$(sha256_hash_in_hex $FILE_CONTENT)"

DATE_TIME=`date -u +%Y%m%dT%H%M%SZ`
DATE=`date -u +%Y%m%d`

# HTTP Headers
HEADER_CONTENT_LENGTH="content-length:$FILE_SIZE"
HEADER_STORAGE_CLASS="x-amz-storage-class:REDUCED_REDUNDANCY"
HEADER_CONTENT_HASH="x-amz-content-sha256:$CONTENT_HASH"
HEADER_DATE_TIME="x-amz-date:$DATE_TIME"
HEADER_HOST="host:cdklab59-ltv-datastorage.s3.eu-west-2.amazonaws.com"

# Request Parameters
HTTP_METHOD="PUT"
REGION_NAME="eu-west-2"
SERVICE_NAME="s3"
CANONICAL_URI="/${FILE_NAME}"
CANONICAL_QUERY_STRING=""

SCHEME="AWS4"
ALGORITHM="HMAC-SHA256"

CANONICAL_HEADERS="${HEADER_CONTENT_LENGTH}\n${HEADER_HOST}\n${HEADER_CONTENT_HASH}\n${HEADER_DATE_TIME}\n${HEADER_STORAGE_CLASS}\n"
CANONICAL_HEADER_NAMES="content-length;host;x-amz-content-sha256;x-amz-date;x-amz-storage-class"

SCOPE="${DATE}/${REGION_NAME}/${SERVICE_NAME}/aws4_request"

function create_hashed_canonical_request() {
   CANONICAL_REQUEST_CONTENT="${HTTP_METHOD}\n${CANONICAL_URI}\n${CANONICAL_QUERY_STRING}\n${CANONICAL_HEADERS}\n${CANONICAL_HEADER_NAMES}\n${CONTENT_HASH}"

   CANONICAL_REQUEST=$(sha256_hash_in_hex "${CANONICAL_REQUEST_CONTENT}")

   printf "$CANONICAL_REQUEST"
}

function create_string_to_sign() {
   STRING_TO_SIGN="${SCHEME}-${ALGORITHM}\n${DATE_TIME}\n${SCOPE}\n$(create_hashed_canonical_request)"

   printf "$STRING_TO_SIGN"
}

function sign() {
 DATE_HMAC=$(hex_of_sha256_hmac_with_string_key_and_value "AWS4${SECRET_ACCESS_KEY}" ${DATE})
 REGION_HMAC=$(hex_of_sha256_hmac_with_hex_key_and_value "${DATE_HMAC}" ${REGION_NAME})
 SERVICE_HMAC=$(hex_of_sha256_hmac_with_hex_key_and_value "${REGION_HMAC}" ${SERVICE_NAME})
 SIGNING_HMAC=$(hex_of_sha256_hmac_with_hex_key_and_value "${SERVICE_HMAC}" "aws4_request")
 SIGNATURE=$(hex_of_sha256_hmac_with_hex_key_and_value "${SIGNING_HMAC}" "$(create_string_to_sign)")

 printf "${SIGNATURE}"
}

function create_authorization_header() {
 printf "${SCHEME}-${ALGORITHM} Credential=${ACCESS_KEY}/${SCOPE}, SignedHeaders=${CANONICAL_HEADER_NAMES}, Signature=$(sign)"
}


HEADER_AUTHORISATION="$(create_authorization_header)"
```
- The request itself
I used curl to make the request because of its maturity and flexibility when it comes to options:
```bash
curl -L -X PUT -T "${FILE_TO_UPLOAD}" --post301 \
	-H "${HEADER_CONTENT_LENGTH}" \
	-H "Host: <account-name>.s3.<region>.amazonaws.com" \
	-H "${HEADER_CONTENT_HASH}" \
	-H "${HEADER_DATE_TIME}" \
	-H "${HEADER_STORAGE_CLASS}" \
	-H "Authorization: ${HEADER_AUTHORISATION}" \
https://<account-name>.s3.<region>.amazonaws.com${CANONICAL_URI}
```
As you can see, I’m including there a number of variables that have been either set in the beginning or calculated with the user of helper functions.
The request makes a PUT call to upload a file into the S3 storage.

I hope this helps. Let me know in the comments if you see any improvements that I could apply to the above.
