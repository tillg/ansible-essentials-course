# Final setup

During the tutorial individual areas are built out chapter by chapter. This repo contains the _end state_ of the course. Here is a description of this setup, including how to launch and inspect it.

## Overview

## Starting the environment

## Access points

## Inner structure

### Web servers

The web servers run Apache2 on Ubuntu. The directory structure (i.e. the relevant parts of it) looks as follows:

```
/ -
	+ etc/
		+ apache2/
			+ apache2.conf # The apache2 config file
			+ conf-available/
			+ conf-enabled/
				+ ansible_site.conf # The config of the ansible site
	+ var/
		+ www/
			+ ansible/
```
