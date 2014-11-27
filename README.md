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
			core_variable_array_append errors "There is the error (number $eventIndex) '$code' in resource '$resource's field '$field'."
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

### Global Constants

|Constant|Value|Explanation|
|--------|-----|-----------|
|`jsonreader_path_index`|`[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]`|Glob expression to match array indices in `jsonreader_path`. See below.|

### Functions

***
#### `jsonreader_parse()`

|Parameter|Value|Optional|
|---------|-----|--------|
|`jsonFilePath`|Path to a file that contains JSON.|_No_|
|`eventCallback`|Name of a function to callback.|_Yes_|

The callback function is passed no arguments, but has the following variables available to it:-

|Variable|Description|
|--------|-----------|
|`eventKind`|The kind of event. One of `object`, `array` or `root` (if this is a solitary value; rare)|
|`eventVariant`|The `start` or `end` of an object or array. The type of value: `boolean` (including null), `number` or `string`|
|`eventValue`|The parent `eventKind` for `start` and `end`. `null`, `true` or `false` for boolean. Number or string. Numbers have exponent casing normalised to `E`|
|`eventIndex`|Field index (zero-based) for `value` in array or object or 0 if value is a `root`. At `start` and `end`, field index in parent (this is logical but not obvious)|
|`eventKey`|Field key for objects. Unset for arrays or if value is a `root`. At `start` and `end`, set to field key in parent if parent is an object.|
|`eventCount`|Only valid for `end`. Count of fields parsed.|
|`jsonreader_path`|A path suitable for globbing representing where in the JSON we are.|

##### `jsonreader_path`
`jsonreader_path` is used to quickly identify depth in the JSON graph. Examples best illustrate its values:-

|Example|Sample JSON|Explanation
|-------|-----------|----------|
|_(empty)_|`true`|Only occurs for a `root` event|
|`/`|`{"key":"value"}`|`start` or `end` of object. `start` has an `eventValue` of `root` and an `eventKind` of `object`.|
|`/key`|`{"key":"value"}`|`string` of key. `eventKey` is `key`, `eventIndex` is 0, `eventVariant` is `string`, `eventKind` is `object`|
|`/key/`|`{"key":{"nested":"value"}}}`|`start` or `end` of nested object. `end` has an `eventCount` of `1`. `start` and `end` have an `eventValue` of `object.`|
|`/key/nested`|`{"key":{"nested":"value"}}}`|`string` of `nested`|
|`:`|`["value"]`|`start` or `end` of array|
|`:0000000005`|`[0,1,2,3,4,5.1e4]`|value of 6th element of the array (`5.1E4`)|
|`:0000000005/`|`[0,1,2,3,4,{}]`|`start` or `end` of nested object. `start` has an `eventValue` of `array` and an `eventKind` of `object`|
|`:0000000005/key`|`[0,1,2,3,4,{"key":"value"}]`|`key` in nested object of the 6th element of the array. `eventValue` is `value`|
|`:0000000005/key:0000000001`|`[0,1,2,3,4,{"key":[0,1]]`|index 1 in nested array of nested object of the 6th element of the array. `eventValue` is `1`|

A `:` is used to denote an array, and a `/` for an object. Object keys are then post-fixed to `/`. Array indices are post-fixed, fixed-width, to `:`. The reason for the weird numbering syntax for array ordinal position is that the fixed width allows use of the shells' glob syntax, rather than a call out to `grep` - the width permits 99,999,999 entries in an array. An empty key in a JSON object would cause a path fragment of `//`.

Matching the `jsonreader_path` can be done using the helper function `jsonreader_matches()`. The glob expression to match array indices can use the constant `jsonreader_path_index`.

***
#### `jsonreader_matches()`

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
[jsonreader]: https://github.com/shellfire-dev/jsonreader "jsonreader shellfire module homepage"
[urlencode]: https://github.com/shellfire-dev/urlencode "urlencode shellfire module homepage"
[unicode]: https://github.com/shellfire-dev/unicode "unicode shellfire module homepage"
[version]: https://github.com/shellfire-dev/version "version shellfire module homepage"