<script>

	//import Global CSS from the svelte boilerplate
	//contains Figma color vars, spacing vars, utility classes and more
    import { GlobalCSS, Input, Label, IconKey, IconHyperlink, Button, IconButton, Radio } from 'figma-plugin-ds-svelte';
    import { loading, winWidth, apiKey, binURL, mainSection, baseURL, themeData, tutorialURL } from '../scripts/stores.js';
    import HeaderGraphic from '../assets/header.svg';
    import IconHelp from '../assets/help.svg';

    let className = '';
    let width = $winWidth + 'px';
    export { className as class };

    let buttonDisabled = true;
    let selectedStorage = 'JSONBin';

    //function to reset Themer
    //this function needs to reset values of key variables
    //and also pass a msg to Figma to clear the API and BIN values
    function resetThemer() {

        //reset onboarding data in Figma api
        parent.postMessage({ pluginMessage: { 'type': 'reset' } }, '*');

    }

    //function to connect to JSONBIn and connect to a bin
    //if the url is empty, we create a new empty bin
    function saveSettings() {

        console.log($binURL);

        //loading state
        $loading = true;

        //if the bin url is empty, we create a new bin
        //this will send an empty json dataset to jsonBin
        if ($binURL === '') {

            //JSON bin needs an empty array for it to be valid
            let data = '[{}]';

            //connect to JSON bin and create the bin
            let req = new XMLHttpRequest();

            //waits for the request
            req.onreadystatechange = () => {
                if (req.readyState == XMLHttpRequest.DONE && [200, 201].includes(req.status)) {

                    let responseData = JSON.parse(req.responseText);

                    if (selectedStorage === 'JSONBin') {
                        //get the data about the new bin
                        let binId = responseData.metadata.id;
                        $binURL = $baseURL + binId;
                        $themeData = responseData.record;
                    } else if (selectedStorage === 'Gist') {
                        $binURL = responseData.url;
                        $themeData = responseData.files['Themer Figma Plugin'].content;
                    }

                    //send the data to figma to save to client storage + success msg
                    parent.postMessage({ pluginMessage: { 'type': 'saveCredentials', 'apiKey': $apiKey, 'binURL': $binURL, 'storage': selectedStorage} }, '*');

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $loading = false;
                    }, 200);

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $mainSection = 'themes';
                    }, 700);


                } else if (req.status >= 400) {

                    //send error message to user
                    parent.postMessage({ pluginMessage: { 'type': 'notify', 'message': `Connection to ${selectedStorage} failed. Double check your API key.`} }, '*');

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $loading = false;
                    }, 200);
                }
            };

            //make the request to JSONBin or Github Gist to create a bin to store theme data
            if (selectedStorage === 'JSONBin') {
                req.open('POST', 'https://api.jsonbin.io/v3/b', true);
                req.setRequestHeader('Content-Type', 'application/json');
                req.setRequestHeader('X-Master-Key', $apiKey);
                req.setRequestHeader('X-Bin-Name', 'Themer Figma Plugin');
                req.send(data);
            }

            if (selectedStorage === 'Gist') {
                data = {
                    description: 'Themer data',
                    'public': false,
                    files: {
                        'Themer Figma Plugin': {
                            content: '[{}]'
                        }
                    }
                }
                req.open('POST', ' https://api.github.com/gists', true);
                req.setRequestHeader('Accept', 'application/vnd.github+json');
                req.setRequestHeader('Content-Type', 'application/json');
                req.setRequestHeader('Authorization', 'Bearer ' + $apiKey);
                req.send(JSON.stringify(data));
            }

        //binurl is present, make sure its a json bin url
        //this could probably be more sophisticated but I am an idiot
        } else if (!$binURL.includes('https://api.jsonbin.io/') && !$binURL.includes('https://gist.github.com/')) { 
            
            //send a message to the user with error state
            parent.postMessage({ pluginMessage: { 'type': 'notify', 'message': 'Invalid bin url.'} }, '*');

            //turn off the loading state with brief delay
            setTimeout(() => {
                $loading = false;
            }, 200);

        //if url is present we just want to connect to it and load the data
        } else {

            //detect if this is an older jsonbin url
            if (selectedStorage === 'JSONBin' && !$binURL.includes('https://api.jsonbin.io/v3/b')) {
                $binURL = $binURL.replace('https://api.jsonbin.io/b','https://api.jsonbin.io/v3/b');

                console.log('new bin url: ', $binURL);
				console.log('old json bin url, migrating to v3');
            }

            let req = new XMLHttpRequest();

            req.onreadystatechange = () => {

                //if the request is successful (1)
                if (req.readyState == XMLHttpRequest.DONE && req.status === 200) {

                    let responseData = JSON.parse(req.responseText);

                    if (selectedStorage === 'Gist') {
                        $themeData = responseData.files['Themer Figma Plugin'].content;
                    } else {
                        $themeData = responseData.record;
                    }

                    //send error message to user
                    parent.postMessage({ pluginMessage: { 'type': 'saveCredentials', 'apiKey': $apiKey, 'binURL': $binURL} }, '*');

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $loading = false;
                    }, 200);

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $mainSection = 'themes';
                    }, 500);
                
                } else if (req.status >= 400) { //if unsuccessful (2)

                    //send user to the settings page
                    $mainSection = 'settings';
                    
                    //send error message to user
                    parent.postMessage({ pluginMessage: { 'type': 'notify', 'message': `Connection to ${selectedStorage} failed. Double check your API key.`} }, '*');

                    //turn off the loading state with brief delay
                    setTimeout(() => {
                        $loading = false;
                    }, 500);

                }
            };

            if (selectedStorage === 'JSONBin') {
                req.open('GET', $binURL + '/latest', true);
                req.setRequestHeader('Content-Type', 'application/json');
                req.setRequestHeader('X-Master-Key', $apiKey);
                req.send();
            }

            if (selectedStorage === 'Gist') {
                req.open('GET', $binURL, true);
                req.setRequestHeader('Content-Type', 'application/vnd.github+json');
                req.setRequestHeader('Authorization', 'Bearer ' + $apiKey);
                req.send();
            }

        }
        
    }

    //launch the video tutorial 
    function videoTutorial() {
        window.open($tutorialURL, '_blank').focus();
    }


</script>

<div class="container flex column {className}" style="width:{width};">

    <div class="header">
        <div class="get-help flex aling-content-center justify-content-center align-items-center">
            <IconButton iconName={IconHelp} on:click={() => videoTutorial() } />
        </div>
        {@html HeaderGraphic}
    </div>

    <div class="flex column flex-grow pt-xsmall pl-xsmall pr-xsmall">
        <div class="flex justify-content-between">
            <div class="flex">
                <Radio bind:group={selectedStorage} value="JSONBin">JSONBin</Radio>
            </div>
            
            <div class="flex">
                <Radio bind:group={selectedStorage} value="Gist">Github Gist</Radio>
            </div>
        </div>

        <div class="label-offset"><Label>API Key</Label></div>
        <Input iconName={IconKey} placeholder="API key from JSONBin.com or gist.github.com" bind:value={$apiKey} class="mb-xxsmall" borders=true />

        <div class="label-offset"><Label>URL to your JSONBin or Github Gist</Label></div>
        <Input iconName={IconHyperlink} bind:value={$binURL} placeholder="Leave empty to auto-generate" borders=true/>
    </div>

    <div class="footer flex pr-xsmall pl-xsmall align-items-center justify-content-between">
        <Button variant='primary' on:click={() => saveSettings()} disabled={$apiKey === ''}>Save</Button>
        <Button variant='tertiary' destructive=true on:click={() => resetThemer()}>Reset Themer</Button>
    </div>

    </div>


<style>

.header {
    position: relative;
    padding-top: 1px;
    box-shadow: 0px 1px 0px var(--figma-color-border);

}

.label-offset {
    margin-left: -8px;
}

.get-help {
    display: block;
    position: absolute;
    top: 5px;
    right: 4px;
}

.footer {
    border-top: 1px solid var(--figma-color-border);
    height:var(--size-xlarge);
}

.container {
    width: 100%;
    height: 100%;
}

</style>