<script lang="ts">
    import toast from 'svelte-french-toast';

    import LoadingSpinner from './LoadingSpinner.svelte';
    import imageExternal from '../../assets/ic_external.svg';

    import {uploadAndPin, unpin, getCidFromIpfsUrl} from "../../lib/ipfs-service";
    import {IMAGE_TYPES, imageMimeTypesJoined, MAX_FILE_SIZE, supportedMimeTypesJoined} from '../../lib/file-utils'
    import {attachments, author, cover, description, file, title} from "../../lib/stores/state-store";

    import * as id3 from "id3js";
    import type {Web3File} from "web3.storage";
    import InlineSVG from "svelte-inline-svg";
    import {createEventDispatcher} from "svelte";
    import {INFURA_GATEWAY_URL} from "../../config";

    export let disabled: boolean = false;
    export let isCollectable: boolean;
    export let collectPrice: string;

    $: attachmentCid = $attachments?.[0]?.item.startsWith('ipfs://') ? getCidFromIpfsUrl($attachments[0].item) : undefined;
    $: attachmentPath = attachmentCid ? `${INFURA_GATEWAY_URL}${attachmentCid}` : $attachments?.[0]?.item;
    $: attachmentType = $attachments?.[0]?.type;

    $: coverPath = $cover?.cid ? `${INFURA_GATEWAY_URL}${$cover?.cid}` : undefined;
    $: coverType = $cover?.type;

    $: isAttachmentImage = attachmentType?.startsWith('image/');
    $: isAttachmentAudio = attachmentType?.startsWith('audio/');
    $: isAttachmentVideo = attachmentType?.startsWith('video/');

    let fileInput;
    let coverInput;
    let loading = false;
    let coverLoading = false;
    let uploadedPct = 0;
    let useContentAsDescription = true;
    let isFileDragged = false;

    const dispatch = createEventDispatcher();

    const processId3Tags = async (file: File) => {
        const tags = await id3.fromFile(file);
        if (!tags) return;

        console.log('processId3Tags: got tags from file', tags)
        if (tags.title) {
            title.set(tags.title);
        } else if (file.name) {
            title.set(file.name.replace(/\.[^/.]+$/, ""));
        }

        if (tags.artist) {
            author.set(tags.artist);
        }

        if (tags.images) {
            const imageBuffer = tags.images[0].data;
            const id3Cover = new File([imageBuffer], 'cover.jpeg', {type: tags.images[0].mime});
            await upload(id3Cover, true);
        }
    };

    const upload = async (fileToUpload: Web3File, isCover: boolean = false) => {
        console.log('upload: ', fileToUpload.name);
        if (isCover) {
            coverLoading = true;
        } else {
            loading = true;
        }

        try {
            fileToUpload.cid = await uploadAndPin(fileToUpload, pct => {
                console.log('upload: Uploading...', pct.toFixed(2))
                uploadedPct = pct;
            });
        } catch (e) {
            console.error(e)

            if (isCover) {
                cover.set(null);
                coverLoading = false;
            } else {
                $attachments = null;
                loading = false;
            }

            uploadedPct = 0;
            $file = null;

            toast.error('upload: Error uploading file');

            return;
        }

        console.log('upload: Uploaded file at', fileToUpload.cid);

        if (isCover) {
            cover.set(fileToUpload);
            return;
        }

        if (!$attachments) {
            $attachments = [];
        }

        $attachments[0] = {
            item: 'ipfs://' + fileToUpload.cid,
            type: fileToUpload.type,
        };

        console.log('upload: created attachment', $attachments);

        if (fileToUpload.type.startsWith('audio/')) {
            try {
                await processId3Tags(fileToUpload);
            } catch (e) {
                console.warn('upload: Error loading audio ID3 tag metadata');
            }
        }

        $file = null;
    };

    const onFileSelected = async (e, isCover: boolean = false) => {
        const file = e.target.files[0];
        const maxSize = isCover ? MAX_FILE_SIZE / 10 : MAX_FILE_SIZE;

        if (!file || file.size > maxSize) {
            toast.error(`File too large. Max ${maxSize}MB`);
            return;
        }
        console.log('File selected', file);

        await upload(file, isCover);
    };

    const onDeleteCover = async () => {
        if (!$cover) return;

        if ($cover.cid) {
            try {
                await unpin($cover.cid);
            } catch (e) {
                console.warn('Unable to unpin cid', $cover.cid)
            }
        }

        cover.set(null);
        coverLoading = false;
    };

    const deleteAttachment = async (notify: boolean = false) => {
        if ($attachments?.[0]?.item?.startsWith('ipfs://')) {
            const cid = getCidFromIpfsUrl($attachments?.[0]?.item);
            if (cid) {
                try {
                    await unpin(cid);
                } catch (e) {
                    console.warn('Unable to unpin cid', cid)
                }
            }
        }

        $attachments = null;
        loading = false;

        if (notify) dispatch('attachmentRemoved');
    }

    const onDeleteMedia = async (notify: boolean = false) => {
        await deleteAttachment(notify);
        await onDeleteCover();
        uploadedPct = 0;
    };

    const onCoverFileDropped = async (ev) => {
        const dt = ev.dataTransfer;
        const file: File = dt.files[0];
        console.log('Cover file dropped', file);

        if (!file.type ||
            !IMAGE_TYPES.includes(file.type) ||
            file.size > (MAX_FILE_SIZE / 10)
        ) {
            toast.error('File not supported');
            isFileDragged = false;
            return;
        }

        cover.set(file);
        isFileDragged = false;

        await onDeleteCover();
        await upload(file, true);
    };

    const onAttachmentLoaded = () => {
        loading = false
        dispatch('attachmentLoaded')
    };

    $: {
        console.log('reactive: file=',$file);
        if ($file && !$file.cid) {
            onDeleteMedia().catch();
            upload($file).catch();
        }
    }
</script>

<div class="min-h-48 w-full flex flex-col justify-center items-center gap-4 pb-2">

  {#if attachmentPath && attachmentType}

    <div class="flex w-full justify-center bg-gray-100 dark:bg-gray-700 px-4 pt-6 pb-4 rounded-xl">

      <div class="flex flex-col items-center justify-center {isCollectable ? 'w-5/12' : 'w-full'} ">
        {#if isAttachmentImage}

          {#if loading}
            <LoadingSpinner message="Processing..."/>
          {/if}

          <div class="relative">
            <img src={attachmentPath} alt="Uploaded file" class="max-w-full max-h-96 rounded-xl" crossorigin
                 on:load={onAttachmentLoaded}>

            <div class="absolute flex justify-end items-start top-0 left-0 z-10 w-full h-full group">
              <button type="button" class="mt-2 mr-2 hidden group-hover:block" on:click={() => onDeleteMedia(true)}>
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="currentColor"
                     class="w-8 h-8 text-white drop-shadow-dark">
                  <!--! Font Awesome Pro 6.2.0 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license (Commercial License) Copyright 2022 Fonticons, Inc.-->
                  <path
                      d="M256 512c141.4 0 256-114.6 256-256S397.4 0 256 0 0 114.6 0 256s114.6 256 256 256zm-81-337c9.4-9.4 24.6-9.4 33.9 0l47 47 47-47c9.4-9.4 24.6-9.4 33.9 0s9.4 24.6 0 33.9l-47 47 47 47c9.4 9.4 9.4 24.6 0 33.9s-24.6 9.4-33.9 0l-47-47-47 47c-9.4 9.4-24.6 9.4-33.9 0s-9.4-24.6 0-33.9l47-47-47-47c-9.4-9.4-9.4-24.6 0-33.9z"/>
                </svg>
              </button>
            </div>

          </div>

        {:else}

          <div class="w-full flex {isCollectable ? 'flex-col' : 'flex-row gap-4'} items-center">

            <div class="{isCollectable ? 'w-full' : 'w-2/5'} mb-4 rounded-xl {isAttachmentVideo ? 'p-0' : 'p-1'}
                 {isFileDragged ? 'bg-orange-50 dark:bg-gray-800' : 'bg-none'}"
                 on:drop|preventDefault|stopPropagation={onCoverFileDropped}
                 on:dragenter|preventDefault|stopPropagation={() => isFileDragged = true}
                 on:dragover|preventDefault|stopPropagation={() => isFileDragged = true}
                 on:dragleave|preventDefault|stopPropagation={() => isFileDragged = false}>

              {#if coverPath}

                <div class="w-full relative bg-gray-200 rounded-xl flex justify-center items-center
                     {isAttachmentVideo ? 'aspect-video' : ''}">
                  <img src={coverPath} alt="Cover" crossorigin
                       class="w-full h-full bg-cover rounded-xl
                             {isFileDragged ? 'border border-orange-500' : 'border-none'}"
                       on:load={() => coverLoading = false}>

                  <div class="absolute flex justify-end items-start top-0 left-0 z-10 w-full h-full group">
                    <button type="button" class="mt-2 mr-2 hidden group-hover:block" on:click={onDeleteCover}>
                      <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" fill="currentColor"
                           class="w-8 h-8 text-white drop-shadow-dark">
                        <!--! Font Awesome Pro 6.2.0 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license (Commercial License) Copyright 2022 Fonticons, Inc.-->
                        <path
                            d="M256 512c141.4 0 256-114.6 256-256S397.4 0 256 0 0 114.6 0 256s114.6 256 256 256zm-81-337c9.4-9.4 24.6-9.4 33.9 0l47 47 47-47c9.4-9.4 24.6-9.4 33.9 0s9.4 24.6 0 33.9l-47 47 47 47c9.4 9.4 9.4 24.6 0 33.9s-24.6 9.4-33.9 0l-47-47-47 47c-9.4 9.4-24.6 9.4-33.9 0s-9.4-24.6 0-33.9l47-47-47-47c-9.4-9.4-9.4-24.6 0-33.9z"/>
                      </svg>
                    </button>
                  </div>

                </div>

              {:else if coverLoading}

                <div class="{isAttachmentVideo ? 'aspect-video' : 'aspect-square'} flex flex-col items-center justify-center">
                  <LoadingSpinner />
                </div>

              {:else}

                <input type="file" class="hidden"
                       accept={imageMimeTypesJoined()}
                       on:change={e => onFileSelected(e, true)}
                       bind:this={coverInput}/>

                <button type="button" class="{isAttachmentVideo ? 'aspect-video' : 'aspect-square'} w-full flex flex-col
                        flex-shrink-0 items-center justify-center border border-gray-300 dark:border-gray-500 text-gray-300
                        dark:text-gray-300 rounded-xl cursor-pointer hover:bg-orange-50"
                        on:click={()=>{coverInput.click();}} disabled={coverLoading}>

                  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" class="w-10 text-gray-400"
                       stroke="currentColor" stroke-width="1" stroke-linecap="round" stroke-linejoin="round">
                    <rect x="3" y="3" width="18" height="18" rx="2"/>
                    <circle cx="8.5" cy="8.5" r="1.5"/>
                    <path d="M20.4 14.5L16 10 4 20"/>
                  </svg>

                  <span class="pt-3 text-base">Add a cover image</span>

                </button>

              {/if}

            </div>

            {#if isAttachmentAudio}
              <audio src={attachmentPath} type={attachmentType} crossorigin
                     class="border border-gray-300 rounded-full {!isCollectable ? 'w-full ml-4' : ''}"
                     on:load={() => loading = false}
                     preload="metadata" controls controlslist="nodownload"></audio>
            {:else if isAttachmentVideo}
              <!-- svelte-ignore a11y-media-has-caption -->
              <video src={attachmentPath} type={attachmentType} class="rounded-xl {isCollectable? 'w-full' : 'w-3/5'}" crossorigin
                     on:load={() => loading = false}
                     preload="metadata" controls controlslist="nodownload"></video>
            {/if}
          </div>

        {/if}


        {#if attachmentCid}
          <div class="w-full text-xs text-gray-400 pt-4 px-1 flex gap-6 truncate justify-center">

            <div class="flex truncate">

              <span class="font-semibold">IPFS:</span>

              <div class="flex truncate pl-1">
                <a href={attachmentPath} class="max-w-[10rem] hover:text-gray-600 grow truncate flex" target="_blank"
                   rel="noreferrer">
                  <div class="truncate">{attachmentCid}</div>
                  <InlineSVG src={imageExternal} class="w-3 h-3 flex-shrink-0 inline mt-[0.1rem]"/>
                </a>
              </div>

            </div>

            <button type="button" class="text-red-700 dark:text-white opacity-60 hover:opacity-100"
                    on:click={() => onDeleteMedia(true)}>
              <svg xmlns="http://www.w3.org/2000/svg" class="h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor"
                   stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <polyline points="3 6 5 6 21 6"></polyline>
                <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                <line x1="10" y1="11" x2="10" y2="17"></line>
                <line x1="14" y1="11" x2="14" y2="17"></line>
              </svg>
            </button>

          </div>
        {/if}
      </div>

      {#if isCollectable}

        <div class="flex flex-col grow gap-3 pl-6">

          {#if collectPrice}

            <div class="flex flex-col">
              <div class="text-2xl font-semibold dark:text-gray-100">
                {collectPrice}
              </div>

              <div class="text-sm text-gray-400">
                PRICE
              </div>
            </div>

          {/if}

          <input type="text" placeholder="Title"
                 class="w-full rounded-lg border-gray-200 text-lg placeholder-gray-400 focus:outline-none focus:ring-2
                 focus:ring-orange-200 focus:border-transparent mt-2 disabled:opacity-40
                 dark:bg-gray-600 dark:border-gray-600 dark:text-gray-100"
                 bind:value={$title} disabled={disabled}>

          {#if isAttachmentAudio}
            <input type="text" placeholder="Author"
                   class="w-full rounded-lg border-gray-200 text-lg placeholder-gray-400 focus:outline-none focus:ring-2
                   focus:ring-orange-200 focus:border-transparent mt-2 disabled:opacity-40
                   dark:bg-gray-600 dark:border-gray-600 dark:text-gray-100"
                   bind:value={$author} disabled={disabled}>
          {/if}

          <textarea placeholder="Description (optional)" rows={useContentAsDescription ? '3' : '5'}
                    class="mt-1 rounded-lg border-gray-200 placeholder-gray-400 focus:outline-none focus:ring-2
                    focus:ring-orange-200 focus:border-transparent disabled:opacity-40 resize-none
                    dark:bg-gray-600 dark:border-gray-600 dark:text-gray-100"
                    bind:value={$description} disabled={disabled || useContentAsDescription}></textarea>

          <div class="flex items-center px-3">
            <input id="desc-toggle" name="desc-toggle" type="checkbox"
                   class="w-4 h-4 text-orange-600 dark:text-orange-500 border-gray-200 rounded focus:ring-orange-500"
                   bind:checked={useContentAsDescription} disabled={disabled}>
            <label for="desc-toggle" class="block ml-2 text-sm text-gray-600 dark:text-gray-300"> Use comment as description </label>
          </div>

          <div class="text-xs text-gray-400 px-3">
            Title{isAttachmentAudio ? ', artist, ' : ' '}and description are used on NFT marketplaces but
            may not be shown on all Lens apps.
          </div>

        </div>

      {/if}

    </div>

  {:else if loading && uploadedPct === 0}

    <div class="py-4">
      <LoadingSpinner/>
    </div>

  {:else if uploadedPct > 0 && uploadedPct < 100}

    <div class="flex flex-col gap-2 justify-center items-center">

      <div class="w-72 p-4 m-auto">

        <div class="w-full h-4 bg-gray-400 rounded-full mt-3">
          <div class="h-full text-center text-xs text-white bg-green-500 rounded-full"
               style="width: {uploadedPct}%">
          </div>
        </div>

      </div>

      <div>Uploading...</div>

    </div>

  {:else}

    <div class="h-48 flex justify-center items-center">

      <div class="text-lg pt-4 dark:text-gray-100">
        Drag and drop media or

        <input type="file" class="hidden"
               accept={supportedMimeTypesJoined()}
               on:change={(e)=>onFileSelected(e)}
               bind:this={fileInput}/>

        <button type="button" class="text-base font-semibold text-orange-700 border border-gray-300 rounded-full px-4 py-2 ml-2
                       hover:bg-orange-600 hover:text-white hover:border-transparent dark:text-orange-100 dark:border-gray-600"
                on:click={()=>{fileInput.click();}}>
          Choose a file
        </button>
      </div>

    </div>

  {/if}

</div>