# `jsonreader`: functions module for [shellfire]

This module provides a simple framework for parsing JSON with a [shellfire] application. Since JSON and shell script don't mix easily, we user an event-based parser with callbacks. In _any POSIX shell_ - not just bash! This allows handling of JSON of arbitrary complexity, as long as:-

* There are no empty keys;
* There isn't ascii NUL once decoded

These are both highly unlikely.

## Overview

Usage couldn't be simpler:-

```bash
jsonreader_parse /path/to/file my_callback
```

where `my_callback` is a function that accepts JSON parse events:-

```bash
my_callback()
{
	echo "Depth $jsonreader_path"
	echo "$eventKind"
	echo "$eventVariant"
	echo "$eventValue"
}
```

There is a helper function, `jsonreader_matches()`, to make it easier to distinguish events. For example, the following real logic is taken from the [github api] module for handling arrays of error message objects:-

```bash
_github_api_v3_errorMessageExit()
{
	local errorMessage=''
	local requestId=''
	
	local resource=''
	local code=''
	local field=''
	
	local errors
	local errors_initialised
	core_variable_array_initialise errors
	
	# eg {"message":"Validation Failed","request_id":"d4cb5605-73cd-11e4-85aa-d38119fad9f5","documentation_url":"https://developer.github.com/v3","errors":[{"resource":"ReleaseAsset","code":"already_exists","field":"name"}]}
	_github_api_v3_errorMessage_event()
	{
		if jsonreader_eventMatches '/message' string; then
			errorMessage="$eventValue"
			return 0
		fi
		
		if jsonreader_eventMatches '/request_id' string; then
			requestId="$eventValue"
			return 0
		fi
		
		if jsonreader_eventMatches "/errors:${jsonreader_path_index}/" start; then
			resource=''
			code=''
			field=''
		fi
		
		if jsonreader_eventMatches "/errors:${jsonreader_path_index}/resource" string; then
			resource="$eventValue"
		fi
		
		if jsonreader_eventMatches "/errors:${jsonreader_path_index}/code" string; then
			code="$eventValue"
		fi
		
		if jsonreader_eventMatches "/errors:${jsonreader_path_index}/field" string; then
			field="$eventValue"
		fi
		
		if jsonreader_eventMatches "/errors:${jsonreader_path_index}/" end; then
			core_variable_array_append errors "There is the error (number $arrayIndex) '$code' in resource '$resource's field '$field'."
		fi
	}
	
	jsonreader_parse "$github_api_v3_responseFilePath" _github_api_v3_errorMessage_event
	
	core_exitError $core_commandLine_exitCode_DATAERR "GitHub errors for request '$requestId' ($curl_httpStatusCode) was '$errorMessage' (Details: $(core_variable_array_string errors " "))"
```

This logic _does not run in a subshell_ and so can change variables, mutate state or event exit the running program.

## Importing

To import this module, add a git submodule to your repository. From the root of your git repository in the terminal, type:-
```bash
mkdir -p lib/shellfire
cd lib/shellfire
git submodule add "https://github.com/shellfire-dev/jsonreader.git"
cd -
git submodule init --update
```

You may need to change the url `https://github.com/shellfire-dev/jsonreader.git` above if using a fork.

You will also need to add paths - include the module [paths.d].

You will also need to import the [unicode] module.


## Namespace `jsonreader`

This namespace contains the core functionality needed to access the API.

### To use in code

If calling from another [shellfire] module, add to your shell code the line
```bash
core_usesIn jsonreader
```
in the global scope (ie outside of any functions). A good convention is to put it above any function that depends on functions in this module. If using it directly in a program, put this line _inside_ the `_program()` function:-

```bash
_program()
{
	core_usesIn jsonreader
	…
}
```

### Functions

***
#### `jsonreader_parse`

|Parameter|Value|Optional|
|---------|-----|--------|
|`jsonFilePath`|Path to a file that contains JSON.|_No_|
|`_jsonreader_eventCallback`|Name of a function to callback.|_Yes_|

The callback function is passed no arguments, but has the variables `jsonreader_path`, `eventKind`, `eventVariant` and `eventValue` set. Additionally, depending on context, the variables `arrayIndex`, `objectIndex` and `key` may be set.

##### Event Kinds

|`eventKind`|`eventVariant`|`eventValue` Example|`jsonreader_path` Example|
|-----------|--------------|--------------------|-------------------------|
|`object`|`start`|_Always empty_|`/`, `/author/`. Always ends in `/`.|
|`object`|`end`|`5` - count of fields|`/`, `/author/`. Always ends in `/`.|
|`array`|`start`|_Always empty_|`:`, `/author:`. Always ends in `:`.|
|`array`|`end`|`5` - count of fields|`:`, `/author:`. Always ends in `:`.|
|`value`|`string`|`hello world` - value of string|`:0000000005`, `/string`. Always ends in either the ordinal position of the field (zero-based) or the name of the field. If `object start` has happened, then `key` is the field name and `objectIndex` is the ordinal position. If `array start` has happened, then `arrayIndex` is the ordinal position. If there is a parent object or array, then the parent's ordinal position or key might be accessible.|
|`value`|`number`|`-3.14E05` - value of number. Exponents normalised to `E`.|`:0000000005`, `/string`. As above for string.|
|`value`|`true`|`true`|`:0000000005`, `/string`. As above for string.|
|`value`|`false`|`false`|`:0000000005`, `/string`. As above for string.|
|`value`|`null`|`null`|`:0000000005`, `/string`. As above for string.|

##### `jsonreader_path`
`jsonreader_path` is used to quickly identify depth in the JSON graph. A `:` is used to denote an array, and a `/` for an object. Object keys are then post-fixed to `/`. Array indices are post-fixed, fixed-width, to `:`. The reason for the weird numbering syntax for array ordinal position is that the fixed width allows use of the shells' glob syntax, rather than a call out to `grep` - the width permits 99,999,999 entries in an array.

Matching the `jsonreader_path` can be done using the helper function `jsonreader_matches()`. 

***
#### `jsonreader_matches`

|Parameter|Value|Optional|
|---------|-----|--------|
|`pathGlob`|Glob expression used to match `jsonreader_path`|_No_|
|`variant`|Matched exactly to `eventVariant`|_No_|

Helper function to match events. Use the value `jsonreader_path_index` to glob-match array indices, eg to match `/array:0000000005/hello`, use `/array:${jsonreader_path_index}/hello`.


[swaddle]: https://github.com/raphaelcohn/swaddle "Swaddle homepage"
[shellfire]: https://github.com/shellfire-dev "shellfire homepage"
[core]: https://github.com/shellfire-dev/core "shellfire core module homepage"
[paths.d]: https://github.com/shellfire-dev/paths.d "paths.d shellfire module homepage"
[github api]: https://github.com/shellfire-dev/github "github shellfire module homepage"
[jsonwriter]: https://github.com/shellfire-dev/jsonwriter "jsonwriter shellfire module homepage"
[jsonreader]: https://github.com/shellfire-dev/jsonwriter "jsonreader shellfire module homepage"
[urlencode]: https://github.com/shellfire-dev/jsonwriter "urlencode shellfire module homepage"
