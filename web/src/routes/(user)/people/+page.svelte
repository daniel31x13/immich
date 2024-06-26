<script lang="ts">
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';
  import ImageThumbnail from '$lib/components/assets/thumbnail/image-thumbnail.svelte';
  import Button from '$lib/components/elements/buttons/button.svelte';
  import Icon from '$lib/components/elements/icon.svelte';
  import MergeSuggestionModal from '$lib/components/faces-page/merge-suggestion-modal.svelte';
  import PeopleCard from '$lib/components/faces-page/people-card.svelte';
  import SetBirthDateModal from '$lib/components/faces-page/set-birth-date-modal.svelte';
  import ShowHide, { ToggleVisibilty } from '$lib/components/faces-page/show-hide.svelte';
  import UserPageLayout from '$lib/components/layouts/user-page-layout.svelte';
  import FullScreenModal from '$lib/components/shared-components/full-screen-modal.svelte';
  import {
    notificationController,
    NotificationType,
  } from '$lib/components/shared-components/notification/notification';
  import { ActionQueryParameterValue, AppRoute, QueryParameter } from '$lib/constants';
  import { getPeopleThumbnailUrl } from '$lib/utils';
  import { handleError } from '$lib/utils/handle-error';
  import { shortcut } from '$lib/actions/shortcut';
  import {
    getPerson,
    mergePerson,
    searchPerson,
    updatePeople,
    updatePerson,
    type PeopleUpdateItem,
    type PersonResponseDto,
  } from '@immich/sdk';
  import { mdiAccountOff, mdiEyeOutline } from '@mdi/js';
  import { onMount } from 'svelte';
  import type { PageData } from './$types';
  import { locale } from '$lib/stores/preferences.store';
  import { clearQueryParam } from '$lib/utils/navigation';
  import SearchPeople from '$lib/components/faces-page/people-search.svelte';
  import LinkButton from '$lib/components/elements/buttons/link-button.svelte';
  import { t } from 'svelte-i18n';

  export let data: PageData;

  let people = data.people.people;
  let countTotalPeople = data.people.total;
  let countHiddenPeople = data.people.hidden;

  let selectHidden = false;
  let initialHiddenValues: Record<string, boolean> = {};
  let eyeColorMap: Record<string, 'black' | 'white'> = {};

  let searchName = '';

  let showLoadingSpinner = false;
  let toggleVisibility: ToggleVisibilty = ToggleVisibilty.VIEW_ALL;

  let showChangeNameModal = false;
  let showSetBirthDateModal = false;
  let showMergeModal = false;
  let personName = '';
  let personMerge1: PersonResponseDto;
  let personMerge2: PersonResponseDto;
  let potentialMergePeople: PersonResponseDto[] = [];
  let edittingPerson: PersonResponseDto | null = null;
  let searchedPeopleLocal: PersonResponseDto[] = [];
  let handleSearchPeople: (force?: boolean, name?: string) => Promise<void>;
  let showPeople: PersonResponseDto[] = [];
  let countVisiblePeople: number;
  let changeNameInputEl: HTMLInputElement | null;
  let innerHeight: number;

  for (const person of people) {
    initialHiddenValues[person.id] = person.isHidden;
  }
  $: {
    if (searchName) {
      showPeople = searchedPeopleLocal;
      countVisiblePeople = searchedPeopleLocal.length;
    } else {
      showPeople = people.filter((person) => !person.isHidden);
      countVisiblePeople = countTotalPeople - countHiddenPeople;
    }
  }

  onMount(async () => {
    const getSearchedPeople = $page.url.searchParams.get(QueryParameter.SEARCHED_PEOPLE);
    if (getSearchedPeople) {
      searchName = getSearchedPeople;
      await handleSearchPeople(true, searchName);
    }
  });

  const handleSearch = async () => {
    const getSearchedPeople = $page.url.searchParams.get(QueryParameter.SEARCHED_PEOPLE);
    if (getSearchedPeople !== searchName) {
      $page.url.searchParams.set(QueryParameter.SEARCHED_PEOPLE, searchName);
      await goto($page.url, { keepFocus: true });
    }
  };

  const handleCloseClick = () => {
    for (const person of people) {
      person.isHidden = initialHiddenValues[person.id];
    }
    // trigger reactivity
    people = people;

    // Reset variables used on the "Show & hide people"   modal
    showLoadingSpinner = false;
    selectHidden = false;
    toggleVisibility = ToggleVisibilty.VIEW_ALL;
  };

  const handleResetVisibility = () => {
    for (const person of people) {
      person.isHidden = initialHiddenValues[person.id];
    }

    // trigger reactivity
    people = people;
  };

  const handleToggleVisibility = (toggleVisibility: ToggleVisibilty) => {
    for (const person of people) {
      if (toggleVisibility == ToggleVisibilty.HIDE_ALL) {
        person.isHidden = true;
      }
      if (toggleVisibility == ToggleVisibilty.VIEW_ALL) {
        person.isHidden = false;
      }
      if (toggleVisibility == ToggleVisibilty.HIDE_UNNANEMD && !person.name) {
        person.isHidden = true;
      }
    }

    // trigger reactivity
    people = people;
  };

  const handleDoneClick = async () => {
    showLoadingSpinner = true;
    let changed: PeopleUpdateItem[] = [];
    try {
      // Check if the visibility for each person has been changed
      for (const person of people) {
        if (person.isHidden !== initialHiddenValues[person.id]) {
          changed.push({ id: person.id, isHidden: person.isHidden });
          if (person.isHidden) {
            countHiddenPeople++;
          } else {
            countHiddenPeople--;
          }

          // Update the initial hidden values
          initialHiddenValues[person.id] = person.isHidden;
        }
      }

      if (changed.length > 0) {
        const results = await updatePeople({
          peopleUpdateDto: { people: changed },
        });
        const count = results.filter(({ success }) => success).length;
        if (results.length - count > 0) {
          notificationController.show({
            type: NotificationType.Error,
            message: `Unable to change the visibility for ${results.length - count} ${
              results.length - count <= 1 ? 'person' : 'people'
            }`,
          });
        }
        notificationController.show({
          type: NotificationType.Info,
          message: `Visibility changed for ${count} ${count <= 1 ? 'person' : 'people'}`,
        });
      }
    } catch (error) {
      handleError(
        error,
        `Unable to change the visibility for ${changed.length} ${changed.length <= 1 ? 'person' : 'people'}`,
      );
    }
    // Reset variables used on the "Show & hide people" modal
    showLoadingSpinner = false;
    selectHidden = false;
    toggleVisibility = ToggleVisibilty.VIEW_ALL;
  };

  const handleMergeSamePerson = async (response: [PersonResponseDto, PersonResponseDto]) => {
    const [personToMerge, personToBeMergedIn] = response;
    showMergeModal = false;

    if (!edittingPerson) {
      return;
    }
    try {
      await mergePerson({
        id: personToBeMergedIn.id,
        mergePersonDto: { ids: [personToMerge.id] },
      });

      const mergedPerson = await getPerson({ id: personToBeMergedIn.id });

      people = people.filter((person: PersonResponseDto) => person.id !== personToMerge.id);
      people = people.map((person: PersonResponseDto) => (person.id === personToBeMergedIn.id ? mergedPerson : person));
      if (personToMerge.isHidden) {
        countHiddenPeople--;
      }
      countTotalPeople--;
      notificationController.show({
        message: $t('merge_people_successfully'),
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('unable_to_save_name'));
    }
    if (personToBeMergedIn.name !== personName && edittingPerson.id === personToBeMergedIn.id) {
      /*
       *
       * If the user merges one of the suggested people into the person he's editing it, it's merging the suggested person AND renames
       * the person he's editing
       *
       */
      try {
        await updatePerson({ id: personToBeMergedIn.id, personUpdateDto: { name: personName } });

        for (const person of people) {
          if (person.id === personToBeMergedIn.id) {
            person.name = personName;
            break;
          }
        }
        notificationController.show({
          message: $t('change_name_successfully'),
          type: NotificationType.Info,
        });

        // trigger reactivity
        people = people;
      } catch (error) {
        handleError(error, $t('unable_to_save_name'));
      }
    }
  };

  const handleChangeName = (detail: PersonResponseDto) => {
    showChangeNameModal = true;
    personName = detail.name;
    personMerge1 = detail;
    edittingPerson = detail;

    setTimeout(() => changeNameInputEl?.focus(), 100);
  };

  const handleSetBirthDate = (detail: PersonResponseDto) => {
    showSetBirthDateModal = true;
    edittingPerson = detail;
  };

  const handleHidePerson = async (detail: PersonResponseDto) => {
    try {
      const updatedPerson = await updatePerson({
        id: detail.id,
        personUpdateDto: { isHidden: true },
      });

      people = people.map((person: PersonResponseDto) => {
        if (person.id === updatedPerson.id) {
          return updatedPerson;
        }
        return person;
      });

      for (const person of people) {
        initialHiddenValues[person.id] = person.isHidden;
      }

      showChangeNameModal = false;
      countHiddenPeople++;
      notificationController.show({
        message: $t('changed_visibility_successfully'),
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('unable_to_hide_person'));
    }
  };

  const handleMergePeople = async (detail: PersonResponseDto) => {
    await goto(
      `${AppRoute.PEOPLE}/${detail.id}?${QueryParameter.ACTION}=${ActionQueryParameterValue.MERGE}&${QueryParameter.PREVIOUS_ROUTE}=${AppRoute.PEOPLE}`,
    );
  };

  const submitNameChange = async () => {
    potentialMergePeople = [];
    showChangeNameModal = false;
    if (!edittingPerson || personName === edittingPerson.name) {
      return;
    }
    if (personName === '') {
      await changeName();
      return;
    }
    const data = await searchPerson({ name: personName, withHidden: true });

    // We check if another person has the same name as the name entered by the user

    const existingPerson = data.find(
      (person: PersonResponseDto) =>
        person.name.toLowerCase() === personName.toLowerCase() &&
        edittingPerson &&
        person.id !== edittingPerson.id &&
        person.name,
    );
    if (existingPerson) {
      personMerge2 = existingPerson;
      showMergeModal = true;
      potentialMergePeople = people
        .filter(
          (person: PersonResponseDto) =>
            personMerge2.name.toLowerCase() === person.name.toLowerCase() &&
            person.id !== personMerge2.id &&
            person.id !== personMerge1.id &&
            !person.isHidden,
        )
        .slice(0, 3);
      return;
    }
    await changeName();
  };

  const submitBirthDateChange = async (value: string) => {
    showSetBirthDateModal = false;
    if (!edittingPerson || value === edittingPerson.birthDate) {
      return;
    }

    try {
      const updatedPerson = await updatePerson({
        id: edittingPerson.id,
        personUpdateDto: { birthDate: value.length > 0 ? value : null },
      });

      people = people.map((person: PersonResponseDto) => {
        if (person.id === updatedPerson.id) {
          return updatedPerson;
        }
        return person;
      });
      notificationController.show({
        message: 'Date of birth saved successfully',
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('unable_to_save_name'));
    }
  };

  const changeName = async () => {
    showMergeModal = false;
    showChangeNameModal = false;

    if (!edittingPerson) {
      return;
    }
    try {
      const updatedPerson = await updatePerson({
        id: edittingPerson.id,
        personUpdateDto: { name: personName },
      });
      people = people.map((person: PersonResponseDto) => {
        if (person.id === updatedPerson.id) {
          return updatedPerson;
        }
        return person;
      });
      notificationController.show({
        message: $t('change_name_successfully'),
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, $t('unable_to_save_name'));
    }
  };

  const onResetSearchBar = async () => {
    await clearQueryParam(QueryParameter.SEARCHED_PEOPLE, $page.url);
  };
</script>

<svelte:window bind:innerHeight use:shortcut={{ shortcut: { key: 'Escape' }, onShortcut: handleCloseClick }} />

{#if showMergeModal}
  <MergeSuggestionModal
    {personMerge1}
    {personMerge2}
    {potentialMergePeople}
    on:close={() => (showMergeModal = false)}
    on:reject={() => changeName()}
    on:confirm={(event) => handleMergeSamePerson(event.detail)}
  />
{/if}

<UserPageLayout
  title={$t('people')}
  description={countVisiblePeople === 0 && !searchName ? undefined : `(${countVisiblePeople.toLocaleString($locale)})`}
>
  <svelte:fragment slot="buttons">
    {#if countTotalPeople > 0}
      <div class="flex gap-2 items-center justify-center">
        <div class="hidden sm:block">
          <div class="w-40 lg:w-80 h-10">
            <SearchPeople
              type="searchBar"
              placeholder={$t('search_people')}
              onReset={onResetSearchBar}
              onSearch={handleSearch}
              bind:searchName
              bind:searchedPeopleLocal
              bind:handleSearch={handleSearchPeople}
            />
          </div>
        </div>
        <LinkButton on:click={() => (selectHidden = !selectHidden)}>
          <div class="flex flex-wrap place-items-center justify-center gap-x-1 text-sm">
            <Icon path={mdiEyeOutline} size="18" />
            <p class="ml-2">Show & hide people</p>
          </div>
        </LinkButton>
      </div>
    {/if}
  </svelte:fragment>

  {#if countVisiblePeople > 0 && (!searchName || searchedPeopleLocal.length > 0)}
    <div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 xl:grid-cols-7 2xl:grid-cols-9 gap-1">
      {#each showPeople as person, index (person.id)}
        <PeopleCard
          {person}
          preload={index < 20}
          on:change-name={() => handleChangeName(person)}
          on:set-birth-date={() => handleSetBirthDate(person)}
          on:merge-people={() => handleMergePeople(person)}
          on:hide-person={() => handleHidePerson(person)}
        />
      {/each}
    </div>
  {:else}
    <div class="flex min-h-[calc(66vh_-_11rem)] w-full place-content-center items-center dark:text-white">
      <div class="flex flex-col content-center items-center text-center">
        <Icon path={mdiAccountOff} size="3.5em" />
        <p class="mt-5 text-3xl font-medium max-w-lg line-clamp-2 overflow-hidden">
          {`No people${searchName ? ` named "${searchName}"` : ''}`}
        </p>
      </div>
    </div>
  {/if}

  {#if showChangeNameModal}
    <FullScreenModal title={$t('change_name')} onClose={() => (showChangeNameModal = false)}>
      <form on:submit|preventDefault={submitNameChange} autocomplete="off" id="change-name-form">
        <div class="flex flex-col gap-2">
          <label class="immich-form-label" for="name">{$t('name')}</label>
          <input
            class="immich-form-input"
            id="name"
            name="name"
            type="text"
            bind:value={personName}
            bind:this={changeNameInputEl}
          />
        </div>
      </form>
      <svelte:fragment slot="sticky-bottom">
        <Button
          color="gray"
          fullwidth
          on:click={() => {
            showChangeNameModal = false;
          }}>{$t('cancel')}</Button
        >
        <Button type="submit" fullwidth form="change-name-form">{$t('ok')}</Button>
      </svelte:fragment>
    </FullScreenModal>
  {/if}

  {#if showSetBirthDateModal}
    <SetBirthDateModal
      birthDate={edittingPerson?.birthDate ?? ''}
      on:close={() => (showSetBirthDateModal = false)}
      on:updated={(event) => submitBirthDateChange(event.detail)}
    />
  {/if}
</UserPageLayout>
{#if selectHidden}
  <ShowHide
    onDone={handleDoneClick}
    onClose={handleCloseClick}
    onReset={handleResetVisibility}
    onChange={handleToggleVisibility}
    bind:showLoadingSpinner
    bind:toggleVisibility
    {countTotalPeople}
    screenHeight={innerHeight}
  >
    <div class="w-full grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-5 xl:grid-cols-7 2xl:grid-cols-9 gap-1">
      {#each people as person, index (person.id)}
        <button
          type="button"
          class="relative"
          on:click={() => (person.isHidden = !person.isHidden)}
          on:mouseenter={() => (eyeColorMap[person.id] = 'black')}
          on:mouseleave={() => (eyeColorMap[person.id] = 'white')}
        >
          <ImageThumbnail
            preload={searchName !== '' || index < 20}
            bind:hidden={person.isHidden}
            shadow
            url={getPeopleThumbnailUrl(person.id)}
            altText={person.name}
            widthStyle="100%"
            bind:eyeColor={eyeColorMap[person.id]}
          />
          {#if person.name}
            <span class="absolute bottom-2 left-0 w-full select-text px-1 text-center font-medium text-white">
              {person.name}
            </span>
          {/if}
        </button>
      {/each}
    </div>
  </ShowHide>
{/if}
