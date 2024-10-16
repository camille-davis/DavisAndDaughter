# Communications

mqtt from anywhere:
```
mqtt pub  -h [cloud url] -s  -p [port] -u [username] -pw [password] -t testTopic -q 1  -m "$1 `date`"  -r
```

And email can be done with a curl:
```
curl -X POST https://api.forwardemail.net/v1/emails -u [username] -d "from=[my email]" -d "to=[example]" -d "subject=test" -d "text=test-x"
```

And can text message to xxxxx@[carrier]

Anveo will send sms for all but T-Mo:
```
curl -s "https://www.anveo.com/api/v1.asp?apikey=[api key]&action=sms&from=[number 1]&destination=[number 2]&message=[text]"
```

Here is a one line login to the tmated sites:
```
ssh `ssh -p [secret port] root@[frontend 1 hostname]  grep "ssh\ session:"  [tmate file location] | awk '{print $4}'`
```

