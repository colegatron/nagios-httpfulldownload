nagios-httpfulldownload
=======================

A little nagios plugin to monitorize the download of a url and its assets



#!/usr/bin/php
<?php

##############################################################################
#
# Copyright (c) 2014 - Colegatron - es.linkedin.com/pub/iván-gonzález/24/b3a/416/
#
# This software is subject to the provisions of the Gnu Public License,
# Version 2.0 (ZPL).  A copy of the GPL should accompany this distribution.
# THIS SOFTWARE IS PROVIDED "AS IS" AND ANY AND ALL EXPRESS OR IMPLIED
# WARRANTIES ARE DISCLAIMED, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF TITLE, MERCHANTABILITY, AGAINST INFRINGEMENT, AND FITNESS
# FOR A PARTICULAR PURPOSE
#
##############################################################################

define("STATE_OK", 0);
define("STATE_WARN", 1);
define("STATE_CRIT", 2);
define("STATE_UNKNOWN", 3);

function finish( $status, $text ) { echo $text; exit($status); }

$arguments = getopt( "u:w:", array("url","wget") );

if (!isset($arguments["u"]))
    finish(STATE_UNKNOWN, "Url not specified\n");
else
    $cURL = $arguments["u"];

if ( isset($arguments["w"]) )
   define("WGET", $arguments["w"]);
else
   define("WGET", "/usr/bin/wget");

if (!file_exists(WGET))
    finish( STATE_UNKNOWN, WGET." does not exist\n" );


$cTmpFile="/tmp/zxcvasd-1234-".getmypid();
$aOutput = array();
$nRet = 0;

// Because wget does not downloads javascript files, we download first the
// document, extract the url's and add to the list of url's to download
// Remove '//' and preffix '/'s with $cURL
$cExtraURLS = "";
exec("wget -q -O - ".$cURL, $aOut, $nRet);
preg_match_all("/<script.*src=\"([^\s]+)\"/",join("\n",$aOut),$aMatches);
foreach($aMatches[1] as $cMatch) {
    if ( $cMatch[0].$cMatch[1] == "//" ) $cMatch = substr($cMatch,2);
    if ( $cMatch[0] == "/" ) $cMatch = $cURL.$cMatch;
    $cExtraURLS .= " ".$cMatch;
}
$cURLS = $cURL." ".$cExtraURLS;



$cCmd = "wget --page-requisites $cURLS -O $cTmpFile 2&>".$cTmpFile."-out";
exec($cCmd, $aOutput, $nRet);
if ( $nRet >= 127 )
    finish ( STATE_UNKNOWN, "Error when executing wget\n" );

# Content to extract javascript files url's
$cContents = file_get_contents($cTmpFile);

$aOutput = file($cTmpFile."-out");
unlink ($cTmpFile);
unlink ($cTmpFile."-out");

$bIsFirst = true;
$nTotalOk = 0;
$nTotalWarn = 0;
$nTotalCrit = 0;
$nTotalFiles = 0;
$bDocStatus = false;
$cDocStatus = null;

foreach($aOutput as $cLine) {
    if ( preg_match("/^HTTP request sent/",$cLine) ) {
        $aLine = preg_split("/ /", $cLine);
        if ( $aLine[5] == 200 ) {
            $nTotalOk++;
            if ( $bIsFirst ) {
                $bDocStatus = true;
                $bIsFirst = false;
            }
            else {
                $bDocStatus = false;
                $bIsFirst = false;
            }
        }
        if ( $aLine[5] >= 300 && $aLine[5] < 400 )
           $nTotalFiles--;
        if ( $aLine[5] > 400 && $aLine[5] < 500 )
            $nTotalWarn++;
        if ( $aLine[5] == 500 )
            $nTotalCrit++;
        $nTotalFiles++;
    }
}

$cText = null;
$cDocStatus = ( $bDocStatus ? "Doc OK" : "Doc FAILS" );
if ( $nTotalCrit > 0 ) {
    $cText = ( $cDocStatus." but $nTotalCrit dependent file/s got status 500" );
    $nStatus = STATE_CRIT;
}
elseif ( $nTotalWarn > 0 ) {
    $cText = ( $cDocStatus." but $nTotalWarn dependent file/s got status >200 <500" );
    $nStatus = STATE_WARN;
}
else {
    $cText = rtrim($aOutput[count($aOutput)-1]," \n");
    $nStatus = STATE_OK;
}


echo $cText." [ ".$nTotalFiles."T/".$nTotalOk."K/".$nTotalWarn."W/".$nTotalCrit."C ]\n";
exit ($nStatus);
