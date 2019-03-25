---
layout: default
title: XSS helpers
---

## cross-domain XSS test

Include the following URL in a page to test for ability to load remote scripts. If you see the domain "wreet.xyz" in a popup alert, the page is vulnerable. 

[x.h](/t/x.h)

## XSS via SVG test

If you have the ability to manipulate the URL an image is loaded from on a page, you can try to include the following SVG image to test for XSS. 

[xss.svg](/t/xss.svg)

## XSS breakout generator

Tries to generate escape strings to send as input. It's rough, but I've had some luck with it in the past. Look for red boxes on target page as a hint of successful breakout. 

[breakout.html](/t/breakout.html)

# XSS via flash

If you have the ability to manipulate the URL a flash video is loaded from on a page, you can try to include the following SWF to test for XSS.

[xss.swf](/t/xss.swf)
