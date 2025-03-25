### User send email
- user use Rest API and send email via HTTPS request. It has to, from subject, body timestamp etc.
- Request is handled by the web server.
- MetaData stored in RDBMS, attachment is stored in S3. Reference to location in S3 is stored in RDBMS.
- To support text based search we will store email content in elastic search as well as it provides support for text base searching
- Webserver sends the email to queue for processing.
- SMTP server picks the email from queue. Resolve the address of the emial server it needs to be sent and sent it over SMTP protocol
- Attachment are send as base64 encoding.

### User receives email
- SMTP server gets the email.
- SMTP server then Push the message in queue for processing
- 
- Mail processing server picks the message from queue. Do validations on it. Check for spam/phishing/malware.
- The metaData is stored in RDBMS. Attachment is decoded and stored in S3. Elastic search is used as secondary storage here as well to support text based search
- If user is online(have a valid websocket connection open) mail processing server pushes mail in real time to receiver client.
- If user is offline, When its back online it can fetch mails via https 
