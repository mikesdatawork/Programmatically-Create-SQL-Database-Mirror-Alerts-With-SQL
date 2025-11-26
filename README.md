> **Note** — The folder `linguist-samples/` contains tiny real files so GitHub can correctly display all languages used in this repo.  
> The actual content and examples remain in this README.

![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Programmatically Create SQL Database Mirror Alerts With SQL
**Post Date: September 18, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>Here's some sql logic you can throw into any Job. Create Database Mirror Alert Process. Job is designed to check all databases (excluding system level) to ensure they have been configured for Mirroring based on their Create_Date or Restore_Date (which is most recent). It does NOT check Mirror Databases. Only Principals. This can be deployed across any SQL Server Instance that is currently configured to use Database Mirroring. It is safe to deploy on both Primary or Secondary. If databases have not been configured it compiles a list along with the create/restore dates of said databases, and sends an email. Additionally it will ONLY check Database Instances that have Principals configured on them so if you want to create this Job on the Mirror feel free. It won't do anything until a database has been configured as the Principal on the Server. So whenever the Database Instance fails over via 'set partner failover', then the Job will start running.
Here is what the email will look like:

![Create SQL Mirror Alerts]( https://mikesdatawork.files.wordpress.com/2015/09/image0011.jpg "SQL Mirror Alerts")
 
Here is what the Job will do exactly.
1. Check to see if SQL Database Mail has been configured.
2. Configures SQL Database Mail (automatically) if it does not find the configuration. You will need to supply the SMTP Server Name below.
3. Sends a test email once SQL Database Mail has been configured.
4. Checks to see if ALL databases (excluding system databases) has been configured for SQL Database Mirroring.
5. Creates a temporary table to hold database Names, Create_Date, and Restore_Date.
6. Emails Database List.</p>      



## SQL-Logic
```SQL
use msdb;
set nocount on
set ansi_nulls on
set quoted_identifier on
 
-- Configure SQL Database Mail if it's not already configured.
if (select top 1 name from msdb..sysmail_profile) is null
    begin
-- Enable SQL Database Mail
        exec master..sp_configure 'show advanced options',1
        reconfigure;
        exec master..sp_configure 'database mail xps',1
        reconfigure;
 
-- Add a profile
        execute msdb.dbo.sysmail_add_profile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @description        = 'SQLDatabaseMail';
 
-- Add the account names you want to appear in the email message.
        execute msdb.dbo.sysmail_add_account_sp
            @account_name       = 'sqldatabasemail@mydomain.com'
        ,   @email_address      = 'sqldatabasemail@mydomain.com'
        ,   @mailserver_name    = 'MySMTPServerName@MyDomain.com  
        --, @port           = ####  --optional
        --, @enable_ssl     = 1 --optional
        --, @username       ='MySQLDatabaseMailProfile' --optional
        --, @password       ='MyPassword' --optional
 
        -- Adding the account to the profile
        execute msdb.dbo.sysmail_add_profileaccount_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @account_name       = 'sqldatabasemail@mydomain.com'
        ,   @sequence_number    = 1;
 
        -- Give access to new database mail profile (DatabaseMailUserRole)
        execute msdb.dbo.sysmail_add_principalprofile_sp
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @principal_id       = 0
        ,   @is_default     = 1;
 
-- Get Server info for test message
 
        declare @get_basic_server_name              varchar(255)
        declare @get_basic_server_name_and_instance_name        varchar(255)
        declare @basic_test_subject_message             varchar(255)
        declare @basic_test_body_message                varchar(max)
        set @get_basic_server_name              = (select cast(serverproperty('servername') as varchar(255)))
        set @get_basic_server_name_and_instance_name        = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
        set @basic_test_subject_message             = 'Test SMTP email from SQL Server: ' + @get_basic_server_name_and_instance_name
        set @basic_test_body_message                = 'This is a test SMTP email from SQL Server:  ' + @get_basic_server_name_and_instance_name + char(10) + char(10) + 'If you see this.  It''s working perfectly :)'
 
-- Send quick email to confirm email is properly working.
 
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@mydomain.com'
        ,   @subject        = @basic_test_subject_message
        ,   @body           = @basic_test_body_message;
 
        -- Confirm message send
        -- select * from msdb..sysmail_allitems
    end

-- get basic server info.
 
declare @server_name_basic          varchar(255)
declare @server_name_instance_name      varchar(255)
declare @server_time_zone           varchar(255)
set @server_name_basic          = (select cast(serverproperty('servername') as varchar(255)))
set @server_name_instance_name      = (select  replace(cast(serverproperty('servername') as varchar(255)), '\', '   SQL Instance: '))
 

-- set message subject.
declare @message_subject            varchar(255)
set @message_subject            = 'Server: ' + @server_name_instance_name + ' missing Mirror configuration.'
 

-- get databases that have NOT been configured for Database Mirroring
if object_id('tempdb..#created_or_restored') is not null
    drop table #created_or_restored
 
create table #created_or_restored
    (
        [database]  varchar(255)
    ,   [created_on]    varchar(255)
    ,   [restored_on]   varchar(255)
    );
with last_restored as
(
    select
    databasename = sd.[name]
,   sd.[create_date]
 
,   rh.*
,   rownum = row_number() over (partition by sd.name order by rh.[restore_date] desc)
 
    from
    master.sys.databases sd left outer join msdb.dbo.[restorehistory] rh on rh.[destination_database_name] = sd.name
    join master.sys.database_mirroring sdm on sd.database_id = sdm.database_id
    where
    sd.database_id > 4
    and sd.state_desc = 'online'
    and sdm.mirroring_role_desc is null
    --or    sdm.mirroring_role_desc != 'mirror'
)
insert into #created_or_restored
select
    'database'      = upper(databasename)
,   'created_on'        = replace(replace(left(create_date, 19), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, create_date)
,   'restored_on'       = 
                    case
                        when restore_date is not null then replace(replace(left(restore_date, 19), 'AM', 'am'), 'PM', 'pm') + ' ' + datename(dw, create_date)
                        else ''
                    end
from
    [last_restored]
where
    [rownum] = 1
    and databasename not in ('master', 'model', 'msdb', 'tempdb')

-- create conditions for html tables in top and mid sections of email.
 
declare @xml_top            NVARCHAR(MAX)
declare @xml_mid            NVARCHAR(MAX)
declare @body_top           NVARCHAR(MAX)
declare @body_mid           NVARCHAR(MAX)
 

-- set xml top table td's
-- create html table object for: #created_or_restored
set @xml_top = 
    cast(
        (select
            [database]  as 'td'
        ,   ''
        ,   [created_on]    as 'td'
        ,   ''
        ,   [restored_on]   as 'td'
        ,   ''
 
        from  #created_or_restored
        order by [database] asc
        for xml path('tr')
    ,   elements)
    as NVARCHAR(MAX)
        )

-- set xml mid table td's
-- create html table object for...

-- format email
set @body_top =
        '<html>
        <head>
            <style>
                    h1{
                        font-family: sans-serif;
                        font-size: 110%;
                    }
                    h3{
                        font-family: sans-serif;
                        color: black;
                    }
 
                    table, td, tr, th {
                        font-family: sans-serif;
                        border: 1px solid black;
                        border-collapse: collapse;
                    }
                    th {
                        text-align: left;
                        background-color: gray;
                        color: white;
                        padding: 5px;
                    }
 
                    td {
                        padding: 5px;
                    }
            </style>
        </head>
        <body>
        <H3>' + @message_subject + '</H3>
        <h1>The following databases may require Database Mirroring:</h1>
        <table border = 1>
        <tr>
            <th> Database     </th>
            <th> Created_On       </th>
 
            <th> Last_Restored_On </th>
 
        </tr>'
         
set @body_top = @body_top + @xml_top + '</table>'
     
+ '</table>
        <h1>Go to the server by pasting in the following text under: Start-Run, or (Win + R)</h1>
        <h1>mstsc -v:' + @server_name_basic + '</h1>'
 
+ '</body></html>'

-- send email.
if exists(select top 1 mirroring_role_desc from master.sys.database_mirroring where mirroring_role_desc = 'principal')
    begin
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name       = 'SQLDatabaseMailProfile'
        ,   @recipients     = 'SQLJobAlerts@mydomain.com'
        ,   @subject        = @message_subject
        ,   @body           = @body_top
        ,   @body_format        = 'HTML';
 
    end
drop table #created_or_restored
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")
