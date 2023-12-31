<script lang="ts">
  import { geolocationRequesterStatus, interactionVerb } from "./store";
  import ignoreError from "./utilities/ignoreError";
  import setAreaCenterUsingWebGeolocationApi from "./utilities/setAreaCenterUsingWebGeolocationApi";
  import trackEvent from "./utilities/trackEvent";

  const prompt = async () => {
    geolocationRequesterStatus.set('prompted');
    try {
      await setAreaCenterUsingWebGeolocationApi();
      geolocationRequesterStatus.set(null);
      ignoreError(() => localStorage.setItem("lastKnownWebGeolocationPermissionState", 'granted'));
      trackEvent({
        name: "web-geolocation-prompt-flow-complete",
        title: "Web geolocation prompt flow complete",
      });
    } catch (e) {
      if(!(e instanceof GeolocationPositionError)) {
        ignoreError(() => localStorage.removeItem("lastKnownWebGeolocationPermissionState"));
        throw e;
      }
      
      ignoreError(() => localStorage.setItem("lastKnownWebGeolocationPermissionState", 'denied'));
      geolocationRequesterStatus.set(null);
      trackEvent({
        name: "web-geolocation-prompt-rejected",
        title: "Web geolocation prompt rejected",
      });
    }
  };

  const cancel = async () => {
    geolocationRequesterStatus.set(null);
    trackEvent({
      name: `geolocation-requester-cancelled--${$geolocationRequesterStatus}`,
      title: `GeolocationRequester cancelled (status was ${$geolocationRequesterStatus})`,
    });
  };

  const retry = () => {
    ignoreError(() => localStorage.removeItem("lastKnownWebGeolocationPermissionState"));
    // @ts-ignore
    window.location = window.location.href;
  }
</script>

<div class="geolocation-requester {$geolocationRequesterStatus ? `geolocation-requester--${$geolocationRequesterStatus}` : ''}">
  {#if $geolocationRequesterStatus !== null}
    <div class="geolocation-requester__inner">

      {#if $geolocationRequesterStatus === 'pre-prompt'}

        <h2 class="geolocation-requester__heading">Move circle to your current location?</h2>

        <div class="geolocation-requester__instructions">
          <p><i>Back Of Your Hand</i> needs permission to access your GPS location in this web browser.</p>
          <p>To proceed:</p>
          <ol>
            <li>{$interactionVerb} "Continue".</li>
            <li>{$interactionVerb} "Allow" when prompted by your browser.</li>
          </ol>
        </div>
        
        <div class="button-group">
          <button
            class="button--primary"
            on:click={prompt}>Continue</button>
          <button
            on:click={cancel}>Cancel</button>
        </div>

      {:else if $geolocationRequesterStatus === 'prompted'}

        <p>{$interactionVerb} "Allow"</p>

      {:else if $geolocationRequesterStatus === 'denied'}

        <h2 class="geolocation-requester__heading">Move circle to your current location?</h2>

        <div class="geolocation-requester__instructions">
          <p><i>Back Of Your Hand</i> needs permission to access your GPS location in this web browser.</p>
          <p>You previously blocked this. To proceed:<p>
          <ol>
            <li>Reset all permissions for backofyourhand.com in your web browser settings.</li>
            <li>Click "Retry" below.</li>
          </ol>
        </div>

        <div class="button-group">
          <button
            class="button--primary"
            on:click={retry}>Retry</button>
          <button
            on:click={cancel}>Cancel</button>
        </div>

      {/if}

    </div>
  {/if}
</div>

<style>
  .geolocation-requester {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.95);
    color: white;
    z-index: 9999999;

    display: none;
    align-items: center;
    justify-items: center;
    justify-content: center;
  }

  .geolocation-requester--denied,
  .geolocation-requester--pre-prompt,
  .geolocation-requester--prompted {
    display: flex;
  }

  .geolocation-requester__inner {
    max-width: 600px;
    margin: 20px;
  }

  .geolocation-requester__heading {
    font-size: 2em;
  }

  .geolocation-requester__instructions {
    line-height: 1.5;
    margin: 40px 0;
  }

  .geolocation-requester__instructions > * + * {
    margin-top: 20px;
  }

  .geolocation-requester ol {
    padding-left: 20px;
  }

  .geolocation-requester li {
    list-style: decimal;
  }
</style>