+++
date = '2026-05-03T15:13:08-04:00'
draft = true
title = 'Db Changes in Production Draft'
+++

*   Always run the commands / try it on a test environment first. Always!
    
*   Terminal with "background" color different in PROD versus other env
    
*   Have a peer over your shoulder to prevent other mistakes
    
*   if you need to do any kind of bulk updating:
    
    *   First step: first make sure you get the query right (aka first do a select => can i select the correct set of data that i want)
        
*   Make API call instead of manually editing the DB. Can be hard sometimes coz it can requires cookies, authentication, ...
    
*   If there were was no API ...
    
*   Use transactions : BEGIN . Use rollback then in case of mistake.
    
*   Don't do a meaningless change on the weekend.
    
*   Use Admin Delete (that lets say can update one order at a time) "petal-admin" > API > SQL command
