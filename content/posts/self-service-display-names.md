---
title: "Remember that time you wanted all of your Self Service display names to actually match your policy names?"
date: 2019-07-24
tags: ["jamf api", "script", "self service"]
---

{{< notice info >}}                                                                                                                                                   
These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform. 

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}   

{{< notice note >}}                                                                                                                                                   
The Jamf Pro API basic authentication method used in this script is deprecated. See the following links for updated auth methods:

- [Changes to Classic API authentication in Jamf Pro - what you need to know](https://grahamrpugh.com/2024/05/16/jamf-pro-api-authentication.html)
- [How to convert Classic API scripts to use bearer token authentication](https://community.jamf.com/tech-thoughts-180/how-to-convert-classic-api-scripts-to-use-bearer-token-authentication-53426)
- [Jamf Pro API Authentication Training](https://trainingcatalog.jamf.com/jamf-pro-api-authentication)
- [Understanding Jamf Pro API Roles and Clients](https://www.jamf.com/blog/understanding-jamf-pro-api-roles-and-clients/)
- [Jamf API Roles and Clients Documentation](https://learn.jamf.com/en-US/bundle/jamf-pro-documentation-current/page/API_Roles_and_Clients.html)
- [Jamf Pro API Overview - Authentication and Authorization](https://developer.jamf.com/jamf-pro/docs/jamf-pro-api-overview#authentication-and-authorization)

{{< /notice >}} 

<br/>

-----

You may never want this because your policy names match some convention, or your Self Service display names do, or, whatever. This is just a good find & replace technique for labels in Jamf that can be used pretty much anywhere.

```bash
#!/bin/bash


apipsw=''
apiurl=''
apiusr=''


# get the object id for every Policy
policyids=($(/usr/bin/curl -LSs -H "Accept: application/xml" -u "$apiusr:$apipsw" "$apiurl/policies" | /usr/bin/xmllint --xpath "//policies/policy/id/text()" - | /usr/bin/tr '</id>' '')); /bin/sleep 1


# for each Policy check if Self Service is enabled
for x in "${policyids[@]}"
do
	selfservice="$(/usr/bin/curl -LSs -H "Accept: application/xml" -u "$apiusr:$apipsw" "$apiurl/policies/id/$x" | /usr/bin/xmllint --xpath "//policy/self_service/use_for_self_service/text()" -)"; /bin/sleep 1

	# if Self Service is enabled get the Policy Display Name label from Jamf Pro UI
	if [ "$selfservice" = 'true' ]
	then
		echo "id $x is a Self Service Policy"
	
		displayname=$(/usr/bin/curl -LSs -H "Accept: application/xml" -u "$apiusr:$apipsw" "$apiurl/policies/id/$x" | /usr/bin/xmllint --xpath "//policy/general/name/text()" -); /bin/sleep 1
	
		# change the Self Service Policy Display Name to the Jamf Pro Display Name
		/usr/bin/curl -LSs -X PUT -H "Content-type: application/xml" -d "<policy><self_service><self_service_display_name>$displayname</self_service_display_name></self_service></policy>" -u "$apiusr:$apipsw" "$apiurl/policies/id/$x"
	
		echo "Self Service display name set to: $displayname"; echo
	fi
done
```

<br/><small>Source: https://community.jamf.com/general-discussions-2/remember-that-time-you-wanted-all-of-your-self-service-display-names-to-actually-match-your-policy-names-14733</small>
