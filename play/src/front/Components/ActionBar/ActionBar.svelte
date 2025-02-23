<script lang="ts">
    import { requestedScreenSharingState } from "../../Stores/ScreenSharingStore";
    import {
        audioConstraintStore,
        cameraListStore,
        localStreamStore,
        microphoneListStore,
        speakerListStore,
        requestedCameraState,
        requestedMicrophoneState,
        silentStore,
        videoConstraintStore,
        speakerSelectedStore,
    } from "../../Stores/MediaStore";
    import { ChevronDownIcon, ChevronUpIcon, CheckIcon } from "svelte-feather-icons";
    import cameraImg from "../images/camera.png";
    import cameraOffImg from "../images/camera-off.png";
    import microphoneImg from "../images/microphone.png";
    import microphoneOffImg from "../images/microphone-off.png";
    import layoutPresentationImg from "../images/layout-presentation.png";
    import layoutChatImg from "../images/layout-chat.png";
    import bubbleImg from "../images/bubble-talk.png";
    import followImg from "../images/follow.png";
    import lockOpenImg from "../images/lock-opened.png";
    import lockCloseImg from "../images/lock-closed.png";
    import logoRegister from "../images/logo-register-pixel.png";
    import screenshareOn from "../images/screenshare-on.png";
    import screenshareOff from "../images/screenshare-off.png";
    import emojiPickOn from "../images/emoji-on.png";
    import closeImg from "../images/close.png";
    import penImg from "../images/pen.png";
    import hammerImg from "../images/hammer.png";
    import WorkAdventureImg from "../images/icon-workadventure-white.png";
    import { LayoutMode } from "../../WebRtc/LayoutManager";
    import { embedScreenLayoutStore } from "../../Stores/EmbedScreensStore";
    import { followRoleStore, followStateStore, followUsersStore } from "../../Stores/FollowStore";
    import { gameManager } from "../../Phaser/Game/GameManager";
    import { currentPlayerGroupLockStateStore } from "../../Stores/CurrentPlayerGroupStore";
    import { analyticsClient } from "../../Administration/AnalyticsClient";
    import { chatVisibilityStore, chatZoneLiveStore } from "../../Stores/ChatStore";
    import { proximityMeetingStore } from "../../Stores/MyMediaStore";
    import type { MenuItem, TranslatedMenu } from "../../Stores/MenuStore";
    import {
        activeSubMenuStore,
        menuVisiblilityStore,
        inviteUserActivated,
        SubMenusInterface,
        subMenusStore,
        additionnalButtonsMenu,
    } from "../../Stores/MenuStore";
    import type { Emoji } from "../../Stores/EmoteStore";
    import {
        emoteDataStore,
        emoteDataStoreLoading,
        emoteMenuStore,
        emoteMenuSubCurrentEmojiSelectedStore,
        emoteMenuSubStore,
        emoteStore,
    } from "../../Stores/EmoteStore";
    import LL from "../../../i18n/i18n-svelte";
    import { bottomActionBarVisibilityStore } from "../../Stores/BottomActionBarStore";
    import { fly } from "svelte/transition";
    import { isMediaBreakpointUp } from "../../Utils/BreakpointsUtils";
    import { inExternalServiceStore, myCameraStore, myMicrophoneStore } from "../../Stores/MyMediaStore";
    import { mapEditorModeStore } from "../../Stores/MapEditorStore";
    import { iframeListener } from "../../Api/IframeListener";
    import { onDestroy, onMount } from "svelte";
    import type { Unsubscriber } from "svelte/store";
    import { writable } from "svelte/store";
    import { peerStore } from "../../Stores/PeerStore";
    import { StringUtils } from "../../Utils/StringUtils";
    import Tooltip from "../Util/Tooltip.svelte";
    import { modalIframeStore, modalVisibilityStore } from "../../Stores/ModalStore";
    import { userHasAccessToBackOfficeStore } from "../../Stores/GameStore";
    import { AddButtonActionBarEvent } from "../../Api/Events/Ui/ButtonActionBarEvent";

    const menuImg = gameManager.currentStartedRoom?.miniLogo ?? WorkAdventureImg;

    let cameraActive = false;
    let microphoneActive = false;
    let selectedCamera: string | undefined = undefined;
    let selectedMicrophone: string | undefined = undefined;

    function screenSharingClick(): void {
        if ($silentStore) return;
        if ($requestedScreenSharingState === true) {
            requestedScreenSharingState.disableScreenSharing();
        } else {
            requestedScreenSharingState.enableScreenSharing();
        }
    }

    function cameraClick(): void {
        if ($silentStore) return;
        if ($requestedCameraState === true) {
            requestedCameraState.disableWebcam();
        } else {
            requestedCameraState.enableWebcam();
        }
    }

    function microphoneClick(): void {
        if ($silentStore) return;
        if ($requestedMicrophoneState === true) {
            requestedMicrophoneState.disableMicrophone();
        } else {
            requestedMicrophoneState.enableMicrophone();
        }
    }

    function switchLayoutMode() {
        if ($embedScreenLayoutStore === LayoutMode.Presentation) {
            $embedScreenLayoutStore = LayoutMode.VideoChat;
        } else {
            $embedScreenLayoutStore = LayoutMode.Presentation;
        }
    }

    function followClick() {
        switch ($followStateStore) {
            case "off":
                gameManager.getCurrentGameScene().connection?.emitFollowRequest();
                followRoleStore.set("leader");
                followStateStore.set("active");
                break;
            case "requesting":
            case "active":
            case "ending":
                gameManager.getCurrentGameScene().connection?.emitFollowAbort();
                followUsersStore.stopFollowing();
                break;
        }
    }

    function lockClick() {
        gameManager.getCurrentGameScene().connection?.emitLockGroup(!$currentPlayerGroupLockStateStore);
    }

    function toggleChat() {
        if (!$chatVisibilityStore) {
            menuVisiblilityStore.set(false);
            activeSubMenuStore.set(0);
        }
        chatVisibilityStore.set(!$chatVisibilityStore);
    }

    function toggleEmojiPicker() {
        $emoteMenuSubStore == true ? emoteMenuSubStore.closeEmoteMenu() : emoteMenuSubStore.openEmoteMenu();
    }

    function toggleMapEditorMode() {
        mapEditorModeStore.switchMode(!$mapEditorModeStore);
    }

    function clickEmoji(selected?: number) {
        //if open, in edit mode or playing mode
        if ($emoteMenuStore && selected != undefined) {
            //select place to change in emoji sub menu
            emoteMenuSubCurrentEmojiSelectedStore.set(selected);
        } else if (selected != undefined) {
            //get emoji and play it
            let emoji: Emoji | null | undefined = $emoteDataStore.get(selected);
            if (emoji == undefined) {
                return;
            }
            analyticsClient.launchEmote(emoji);
            emoteStore.set(emoji);

            //play UX animation
            focusElement(selected);
        }
    }

    function edit(): void {
        if ($emoteMenuStore) emoteMenuStore.closeEmoteMenu();
        else emoteMenuStore.openEmoteMenu();
    }

    function close(): void {
        emoteMenuStore.closeEmoteMenu();
        emoteMenuSubStore.closeEmoteMenu();
    }

    function focusElement(key: number) {
        if (!$emoteMenuSubStore) {
            return;
        }
        const name: string | undefined = $emoteDataStore.get(key)?.name;
        if (name == undefined) {
            return;
        }
        const element: HTMLElement | null = document.getElementById(`button-${name}`);
        if (element == undefined) {
            return;
        }
        element.focus();
        element.classList.add("focus");

        //blur element after ends of animation
        setTimeout(() => {
            element.blur();
            element.classList.remove("focus");
        }, 2000);
    }

    function onKeyDown(e: KeyboardEvent) {
        let key = null;
        if (e.key === "1" || e.key === "F1") {
            key = 1;
        }
        if (e.key === "2" || e.key === "F2") {
            key = 2;
        }
        if (e.key === "3" || e.key === "F3") {
            key = 3;
        }
        if (e.key === "4" || e.key === "F4") {
            key = 4;
        }
        if (e.key === "5" || e.key === "F5") {
            key = 5;
        }
        if (e.key === "6" || e.key === "F6") {
            key = 6;
        }
        if (!key) {
            return;
        }
        focusElement(key);
        clickEmoji(key);
    }

    function showInvite() {
        modalVisibilityStore.set(false);

        const indexInviteMenu = $subMenusStore.findIndex(
            (menu: MenuItem) => (menu as TranslatedMenu).key === SubMenusInterface.invite
        );
        if (indexInviteMenu === -1) {
            console.error(`Menu key: ${SubMenusInterface.invite} was not founded in subMenusStore: `, $subMenusStore);
            return;
        }
        if ($menuVisiblilityStore && $activeSubMenuStore === indexInviteMenu) {
            menuVisiblilityStore.set(false);
            activeSubMenuStore.set(0);
            return;
        }
        activeSubMenuStore.set(indexInviteMenu);
        menuVisiblilityStore.set(true);

        resetChatVisibility();
        resetModalVisibility();
    }

    function showMenu() {
        const indexInviteMenu = $subMenusStore.findIndex(
            (menu: MenuItem) => (menu as TranslatedMenu).key === SubMenusInterface.profile
        );
        if (indexInviteMenu === -1) {
            console.error(`Menu key: ${SubMenusInterface.profile} was not founded in subMenusStore: `, $subMenusStore);
            return;
        }
        if ($menuVisiblilityStore && $activeSubMenuStore === indexInviteMenu) {
            menuVisiblilityStore.set(false);
            activeSubMenuStore.set(0);
            return;
        }
        activeSubMenuStore.set(indexInviteMenu);
        menuVisiblilityStore.set(true);

        resetChatVisibility();
        resetModalVisibility();
    }

    function openBo() {
        window.open(`https://workadventu.re/admin`, "_blanck");
    }

    /*function register() {
        modalIframeStore.set(
            {
                src: https://workadventu.re/funnel/connection?roomUrl=${window.location.toString()},
                allow: "fullscreen",
                allowApi: true,
                position: "center",
                title: $LL.menu.icon.open.register()
            }
        );

        //resetMenuVisibility();
        //resetChatVisibility();

        window.open("https://workadventu.re/getting-started", "_blank");
    }*/

    function resetModalVisibility() {
        modalVisibilityStore.set(false);
        modalIframeStore.set(null);
    }

    /*function resetMenuVisibility() {
        menuVisiblilityStore.set(false);
        activeSubMenuStore.set(0);
    }*/

    function resetChatVisibility() {
        chatVisibilityStore.set(false);
    }

    function noDrag() {
        return false;
    }

    function selectCamera(deviceId: string) {
        videoConstraintStore.setDeviceId(deviceId);
        cameraActive = false;
    }

    function selectMicrophone(deviceId: string) {
        audioConstraintStore.setDeviceId(deviceId);
        microphoneActive = false;
    }

    function selectSpeaker(deviceId: string) {
        speakerSelectedStore.set(deviceId);
    }

    let subscribers = new Array<Unsubscriber>();
    let totalMessagesToSee = writable<number>(0);
    onMount(() => {
        iframeListener.chatTotalMessagesToSeeStream.subscribe((total) => totalMessagesToSee.set(total));
    });

    onDestroy(() => {
        subscribers.map((subscriber) => subscriber());
        unsubscribeLocalStreamStore();
    });

    let stream: MediaStream | null;
    const unsubscribeLocalStreamStore = localStreamStore.subscribe((value) => {
        if (value.type === "success") {
            stream = value.stream;

            if (stream !== null) {
                const videoTracks = stream.getVideoTracks();
                if (videoTracks.length > 0) {
                    selectedCamera = videoTracks[0].getSettings().deviceId;
                }
                const audioTracks = stream.getAudioTracks();
                if (audioTracks.length > 0) {
                    // set first track
                    selectedMicrophone = audioTracks[0].getSettings().deviceId;

                    // set default speaker selected
                    if ($speakerListStore.length > 0) {
                        speakerSelectedStore.set($speakerListStore[0].deviceId);
                    }
                }
            }
        } else {
            stream = null;
            selectedCamera = undefined;
            selectedMicrophone = undefined;
        }
    });

    const isMobile = isMediaBreakpointUp("md");

    function buttonActionBarTrigger(id: string) {
        const button = $additionnalButtonsMenu.get(id) as AddButtonActionBarEvent;
        return iframeListener.sendButtonActionBarTriggered(button);
    }
</script>

<svelte:window on:keydown={onKeyDown} />

<div
    class="tw-flex tw-justify-center tw-m-auto tw-absolute tw-left-0 tw-right-0 tw-bottom-0"
    class:animated={$bottomActionBarVisibilityStore}
>
    <div class="bottom-action-bar tw-absolute">
        {#if $bottomActionBarVisibilityStore}
            <div
                class="bottom-action-section tw-flex animate"
                id="bubble-menu"
                in:fly={{ y: 70, duration: 100, delay: 200 }}
                out:fly={{ y: 70, duration: 100, delay: 0 }}
                class:tw-translate-x-0={$bottomActionBarVisibilityStore}
                class:translate-right={!$bottomActionBarVisibilityStore}
            >
                <div
                    class="tw-transition-all bottom-action-button"
                    class:disabled={$followStateStore !== "off"}
                    on:click={() => analyticsClient.follow()}
                    on:click={followClick}
                >
                    <Tooltip text={$LL.actionbar.follow()} />

                    <button class:border-top-light={$followStateStore === "active"}>
                        <img draggable="false" src={followImg} style="padding: 2px" alt="Toggle follow" />
                    </button>
                </div>

                <div
                    class="tw-transition-all bottom-action-button"
                    on:click={() => analyticsClient.layoutPresentChange()}
                    on:click={switchLayoutMode}
                >
                    <Tooltip text={$LL.actionbar.layout()} />

                    <button>
                        {#if $embedScreenLayoutStore === LayoutMode.Presentation}
                            <img
                                draggable="false"
                                src={layoutPresentationImg}
                                style="padding: 2px"
                                alt="Switch to mosaic mode"
                            />
                        {:else}
                            <img
                                draggable="false"
                                src={layoutChatImg}
                                style="padding: 2px"
                                alt="Switch to presentation mode"
                            />
                        {/if}
                    </button>
                </div>

                <div
                    class="tw-transition-all bottom-action-button"
                    class:disabled={$currentPlayerGroupLockStateStore}
                    on:click={() => analyticsClient.lockDiscussion()}
                    on:click={lockClick}
                >
                    <Tooltip text={$LL.actionbar.lock()} />

                    <button class:border-top-light={$currentPlayerGroupLockStateStore}>
                        {#if $currentPlayerGroupLockStateStore}
                            <img
                                draggable="false"
                                src={lockCloseImg}
                                style="padding: 2px"
                                alt="Unlock videochat bubble"
                            />
                        {:else}
                            <img draggable="false" src={lockOpenImg} style="padding: 2px" alt="Lock videochat bubble" />
                        {/if}
                    </button>
                </div>

                <div
                    class="tw-transition-all bottom-action-button"
                    on:click={() => analyticsClient.screenSharing()}
                    on:click={screenSharingClick}
                    class:enabled={$requestedScreenSharingState}
                >
                    <Tooltip text={$LL.actionbar.screensharing()} />

                    <button class:border-top-light={$requestedScreenSharingState}>
                        {#if $requestedScreenSharingState && !$silentStore}
                            <img
                                draggable="false"
                                src={screenshareOn}
                                style="padding: 2px;"
                                alt="Stop screen sharing"
                            />
                        {:else}
                            <img
                                draggable="false"
                                src={screenshareOff}
                                style="padding: 2px;"
                                alt="Start screen sharing"
                            />
                        {/if}
                    </button>
                </div>
            </div>
        {/if}

        <div class="tw-flex tw-flex-row base-section animated">
            <div class="bottom-action-section tw-flex tw-flex-initial">
                {#if !$inExternalServiceStore && !$silentStore && $proximityMeetingStore}
                    {#if $myCameraStore}
                        <div
                            class="bottom-action-button tw-relative"
                            on:click={() => analyticsClient.camera()}
                            on:click={cameraClick}
                            class:disabled={!$requestedCameraState || $silentStore}
                        >
                            <Tooltip text={$LL.actionbar.camera()} />

                            <button
                                class="tooltiptext sm:tw-w-56 md:tw-w-96"
                                class:border-top-light={$requestedCameraState}
                            >
                                {#if $requestedCameraState && !$silentStore}
                                    <img
                                        draggable="false"
                                        src={cameraImg}
                                        style="padding: 2px;"
                                        alt="Turn off webcam"
                                    />
                                {:else}
                                    <img
                                        draggable="false"
                                        src={cameraOffImg}
                                        style="padding: 2px;"
                                        alt="Turn on webcam"
                                    />
                                {/if}
                            </button>

                            {#if $requestedCameraState && $cameraListStore.length > 1}
                                <button
                                    class="camera tw-absolute tw-text-light-purple focus:outline-none tw-m-0"
                                    on:click|stopPropagation|preventDefault={() => (cameraActive = !cameraActive)}
                                >
                                    {#if cameraActive}
                                        <ChevronDownIcon size="13" />
                                    {:else}
                                        <ChevronUpIcon size="13" />
                                    {/if}
                                </button>

                                <!-- camera list -->
                                <div
                                    class={`wa-dropdown-menu ${cameraActive ? "" : "tw-invisible"}`}
                                    style="bottom: 15px;right: 0;"
                                    on:mouseleave={() => (cameraActive = false)}
                                >
                                    {#each $cameraListStore as camera}
                                        <span
                                            class="wa-dropdown-item tw-flex"
                                            on:click={() => {
                                                analyticsClient.selectCamera();
                                            }}
                                            on:click|stopPropagation|preventDefault={() =>
                                                selectCamera(camera.deviceId)}
                                        >
                                            {StringUtils.normalizeDeviceName(camera.label)}
                                            {#if selectedCamera === camera.deviceId}
                                                <CheckIcon size="13" class="tw-ml-1" />
                                            {/if}
                                        </span>
                                    {/each}
                                </div>
                            {/if}
                        </div>
                    {/if}

                    {#if $myMicrophoneStore}
                        <div
                            class="bottom-action-button tw-relative"
                            on:click={() => analyticsClient.microphone()}
                            on:click={microphoneClick}
                            class:disabled={!$requestedMicrophoneState || $silentStore}
                        >
                            <Tooltip text={$LL.actionbar.microphone()} />

                            <button class:border-top-light={$requestedMicrophoneState}>
                                {#if $requestedMicrophoneState && !$silentStore}
                                    <img
                                        draggable="false"
                                        src={microphoneImg}
                                        style="padding: 2px;"
                                        alt="Turn off microphone"
                                    />
                                {:else}
                                    <img
                                        draggable="false"
                                        src={microphoneOffImg}
                                        style="padding: 2px;"
                                        alt="Turn on microphone"
                                    />
                                {/if}
                            </button>

                            {#if $requestedMicrophoneState && $microphoneListStore.length > 1}
                                <button
                                    class="microphone tw-absolute tw-text-light-purple focus:outline-none tw-m-0"
                                    on:click|stopPropagation|preventDefault={() =>
                                        (microphoneActive = !microphoneActive)}
                                >
                                    {#if microphoneActive}
                                        <ChevronDownIcon size="13" />
                                    {:else}
                                        <ChevronUpIcon size="13" />
                                    {/if}
                                </button>

                                <div
                                    class={`wa-dropdown-menu ${microphoneActive ? "" : "tw-invisible"}`}
                                    style="bottom: 15px;right: 0;"
                                    on:mouseleave={() => (microphoneActive = false)}
                                >
                                    {#if $microphoneListStore.length > 0}
                                        <!-- microphone list -->
                                        <span class="tw-underline tw-font-bold tw-text-xs tw-p-1"
                                            >{$LL.actionbar.subtitle.microphone()} 🎙️</span
                                        >
                                        {#each $microphoneListStore as microphone}
                                            <span
                                                class="wa-dropdown-item"
                                                on:click={() => {
                                                    analyticsClient.selectMicrophone();
                                                }}
                                                on:click|stopPropagation|preventDefault={() =>
                                                    selectMicrophone(microphone.deviceId)}
                                            >
                                                {StringUtils.normalizeDeviceName(microphone.label)}
                                                {#if selectedMicrophone === microphone.deviceId}
                                                    <CheckIcon size="13" />
                                                {/if}
                                            </span>
                                        {/each}
                                    {/if}

                                    <!-- speaker list -->
                                    {#if $speakerSelectedStore != undefined && $speakerListStore.length > 0}
                                        <span class="tw-underline tw-font-bold tw-text-xs tw-p-1"
                                            >{$LL.actionbar.subtitle.speaker()} 🔈</span
                                        >
                                        {#each $speakerListStore as speaker}
                                            <span
                                                class="wa-dropdown-item"
                                                on:click={() => {
                                                    analyticsClient.selectSpeaker();
                                                }}
                                                on:click|stopPropagation|preventDefault={() =>
                                                    selectSpeaker(speaker.deviceId)}
                                            >
                                                {StringUtils.normalizeDeviceName(speaker.label)}
                                                {#if $speakerSelectedStore === speaker.deviceId}
                                                    <CheckIcon size="13" />
                                                {/if}
                                            </span>
                                        {/each}
                                    {/if}
                                </div>
                            {/if}
                        </div>
                    {/if}
                {/if}

                <div
                    on:click={() => analyticsClient.openedChat()}
                    on:click={toggleChat}
                    class="bottom-action-button tw-relative"
                >
                    <Tooltip text={$LL.actionbar.chat()} />

                    <button class:border-top-light={$chatVisibilityStore} class="chat-btn">
                        <img draggable="false" src={bubbleImg} style="padding: 2px" alt="Toggle chat" />
                    </button>
                    {#if $chatZoneLiveStore || $peerStore.size > 0}
                        <div class="tw-absolute tw-top-1 tw-right-0.5">
                            <span
                                class={`tw-w-4 tw-h-4 ${
                                    $peerStore.size > 0 ? "tw-bg-pop-green" : "tw-bg-pop-red"
                                } tw-block tw-rounded-full tw-absolute tw-top-0 tw-right-0 tw-animate-ping`}
                            />
                            <span
                                class={`tw-w-3 tw-h-3 ${
                                    $peerStore.size > 0 ? "tw-bg-pop-green" : "tw-bg-pop-red"
                                } tw-block tw-rounded-full tw-absolute tw-top-0.5 tw-right-0.5`}
                            />
                        </div>
                    {:else if $totalMessagesToSee > 0}
                        <span
                            class="tw-absolute tw-top-1.5 tw-right-1 tw-items-center tw-justify-center tw-px-1 tw-py-0.5 tw-text-xxs tw-font-bold tw-leading-none tw-text-white tw-bg-pop-red tw-rounded-full"
                        >
                            {$totalMessagesToSee}
                        </span>
                    {/if}
                </div>
                <div on:click={toggleEmojiPicker} class="bottom-action-button">
                    <Tooltip text={$LL.actionbar.emoji()} />

                    <button class:border-top-light={$emoteMenuSubStore}>
                        <img draggable="false" src={emojiPickOn} style="padding: 2px" alt="Toggle emoji picker" />
                    </button>
                </div>
            </div>

            <div class="bottom-action-section tw-flex tw-flex-initial">
                <div
                    on:dragstart|preventDefault={noDrag}
                    on:click={() => analyticsClient.openedMenu()}
                    on:click={showMenu}
                    class="bottom-action-button"
                >
                    <Tooltip text={$LL.actionbar.menu()} />

                    <button id="menuIcon" class:border-top-light={$menuVisiblilityStore}>
                        <img draggable="false" src={menuImg} style="padding: 2px" alt={$LL.menu.icon.open.menu()} />
                    </button>
                </div>
                {#if gameManager.getCurrentGameScene().isMapEditorEnabled()}
                    <div
                        on:dragstart|preventDefault={noDrag}
                        on:click={toggleMapEditorMode}
                        class="bottom-action-button"
                    >
                        <button id="mapEditorIcon" class:border-top-light={$menuVisiblilityStore}>
                            <img draggable="false" src={logoRegister} style="padding: 2px" alt="toggle-map-editor" />
                        </button>
                    </div>
                {/if}
                {#if $userHasAccessToBackOfficeStore}
                    <div
                        on:dragstart|preventDefault={noDrag}
                        on:click={() => analyticsClient.openBackOffice()}
                        on:click={openBo}
                        class="bottom-action-button"
                    >
                        <Tooltip text={$LL.actionbar.bo()} />

                        <button id="mapEditorIcon">
                            <img draggable="false" src={hammerImg} style="padding: 2px" alt="toggle-map-editor" />
                        </button>
                    </div>
                {/if}
            </div>

            {#if $inviteUserActivated}
                <div
                    class="bottom-action-section tw-flex tw-flex-initial"
                    in:fly={{}}
                    on:dragstart|preventDefault={noDrag}
                    on:click={() => analyticsClient.openInvite()}
                    on:click={showInvite}
                >
                    <button
                        class="btn light tw-m-0 tw-font-bold tw-text-xs sm:tw-text-base"
                        id="invite-btn"
                        class:border-top-light={$menuVisiblilityStore}
                    >
                        {$LL.menu.sub.invite()}
                    </button>
                </div>
            {/if}

            <!-- TODO button must displayed by scripting API -->
            <!--
            {#if ENABLE_OPENID && !$userIsConnected && }
                <div
                    class="bottom-action-section tw-flex tw-flex-initial"
                    in:fly={{}}
                    on:dragstart|preventDefault={noDrag}
                    on:click={() => analyticsClient.openRegister()}
                    on:click={register}
                >
                    <button
                        class="btn light tw-m-0 tw-font-bold tw-text-xs sm:tw-text-base"
                        id="register-btn"
                        class:border-top-light={$menuVisiblilityStore}
                    >
                        {$LL.menu.icon.open.register()}
                    </button>
                </div>
            {/if}
            -->
            {#each [...$additionnalButtonsMenu.values()] as button}
                <div
                    class="bottom-action-section tw-flex tw-flex-initial"
                    in:fly={{}}
                    on:dragstart|preventDefault={noDrag}
                    on:click={() => {
                        buttonActionBarTrigger(button.id);
                    }}
                >
                    <button class="btn light tw-m-0 tw-font-bold tw-text-xs sm:tw-text-base" id={button.id}>
                        {button.label}
                    </button>
                </div>
            {/each}
        </div>
    </div>
</div>

{#if $emoteMenuSubStore}
    <div
        class="tw-flex tw-justify-center tw-m-auto tw-absolute tw-left-0 tw-right-0 tw-bottom-0"
        style="margin-bottom: 4.5rem; height: auto;"
    >
        <div class="bottom-action-bar">
            <div class="bottom-action-section tw-flex animate">
                {#each [...$emoteDataStore.keys()] as key}
                    <div class="tw-transition-all bottom-action-button">
                        <button
                            on:click={() => {
                                clickEmoji(key);
                            }}
                            id={`button-${$emoteDataStore.get(key)?.name}`}
                            class="emoji"
                            class:focus={$emoteMenuStore && $emoteMenuSubCurrentEmojiSelectedStore === key}
                        >
                            <img
                                class="emoji"
                                style="padding: 2px"
                                draggable="false"
                                alt={$emoteDataStore.get(key)?.unicode}
                                id={`icon-${$emoteDataStore.get(key)?.name}`}
                                src={$emoteDataStore.get(key)?.url}
                            />
                            {#if !isMobile}
                                <span class="tw-text-white">{key}</span>
                            {/if}
                        </button>
                    </div>
                {/each}

                <div class="tw-transition-all bottom-action-button">
                    <button on:click={() => analyticsClient.editEmote()} on:click|preventDefault={edit}>
                        {#if $emoteDataStoreLoading}
                            <div class="tw-rounded-lg tw-bg-dark tw-text-xs">
                                <!-- loading animation -->
                                <div class="loading-group">
                                    <span class="loading-dot" />
                                    <span class="loading-dot" />
                                    <span class="loading-dot" />
                                </div>
                            </div>
                        {:else}
                            <img
                                draggable="false"
                                src={penImg}
                                style="padding: 2px"
                                alt={$LL.menu.icon.open.openEmoji()}
                            />
                        {/if}
                    </button>
                </div>
                <div class="tw-transition-all bottom-action-button">
                    <button on:click|preventDefault={close}>
                        <img
                            draggable="false"
                            src={closeImg}
                            style="padding: 4px"
                            alt={$LL.menu.icon.open.closeEmoji()}
                        />
                    </button>
                </div>
            </div>
        </div>
    </div>
{/if}
F

<style lang="scss">
    @import "../../style/breakpoints.scss";

    .animated {
        transition-property: transform;
        transition-duration: 0.5s;
    }

    .translate-right {
        transform: translateX(2rem);
    }

    .bottom-action-section {
        .bottom-action-button {
            button.camera,
            button.microphone {
                top: 0;
                width: 20px;
                height: 20px;
                background: none;
                right: 0;
                border-top-left-radius: 0.25rem;
                border-bottom-left-radius: 0.25rem;
                border-top-right-radius: 0.25rem;
                border-bottom-right-radius: 0.25rem;
                color: white;
                padding: 0;
                margin: 0;
                display: block;
                &:hover {
                    background-color: rgb(56 56 74);
                }
            }
        }
    }

    @include media-breakpoint-down(sm) {
        //is equal to tailwind's sm breakpoint
        .translate-right {
            transform: translateX(0);
        }

        .move-menu {
            transform: translateX(-3rem);
        }
    }
</style>
