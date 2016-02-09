### Solved by et0x

A 300 point challenge with the following description: 

> What is the version of Sharepoint running on an US based server with following details:
> ServerName: CEI02 
> Instance Name: Sharepoint. ?

This was solved by using Shodan [http://www.shodanhq.com/search?q=CEI02+sharepoint](http://www.shodanhq.com/search?q=CEI02+sharepoint): 

```
74.93.173.202
Comcast Business Communications
Added on 04.12.2014
United States Jacksonville

74-93-173-202-Jacksonville.hfc.comcastbusiness.net *This was solved by using Shodan:

*ServerName;CEI02;InstanceName;DIGITALTAKEOFF;IsClustered;No;Version;9.00.5000.00;tcp;49866;np;\\CEI02\pipe\MSSQL$DIGITALTAKEOFF\sql\query;;ServerName;CEI02;InstanceName;SBSMONITORING;IsClustered;No;Version;10.50.2500.0;;ServerName;CEI02;InstanceName;SHAREPOINT;IsClustered;No;Version;10.50.2500.0;;
```

The flag is: **10.50.2500.0**

