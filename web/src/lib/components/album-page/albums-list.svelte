<script lang="ts">
  import { goto } from '$app/navigation';
  import AlbumCardGroup from '$lib/components/album-page/album-card-group.svelte';
  import AlbumsTable from '$lib/components/album-page/albums-table.svelte';
  import MenuOption from '$lib/components/shared-components/context-menu/menu-option.svelte';
  import RightClickContextMenu from '$lib/components/shared-components/context-menu/right-click-context-menu.svelte';
  import {
    NotificationType,
    notificationController,
  } from '$lib/components/shared-components/notification/notification';
  import { AppRoute } from '$lib/constants';
  import AlbumEditModal from '$lib/modals/AlbumEditModal.svelte';
  import AlbumShareModal from '$lib/modals/AlbumShareModal.svelte';
  import QrCodeModal from '$lib/modals/QrCodeModal.svelte';
  import SharedLinkCreateModal from '$lib/modals/SharedLinkCreateModal.svelte';
  import {
    AlbumFilter,
    AlbumGroupBy,
    AlbumSortBy,
    AlbumViewMode,
    SortOrder,
    locale,
    type AlbumViewSettings,
  } from '$lib/stores/preferences.store';
  import { user } from '$lib/stores/user.store';
  import { userInteraction } from '$lib/stores/user.svelte';
  import { makeSharedLinkUrl } from '$lib/utils';
  import {
    confirmAlbumDelete,
    getSelectedAlbumGroupOption,
    sortAlbums,
    stringToSortOrder,
    type AlbumGroup,
  } from '$lib/utils/album-utils';
  import { downloadAlbum } from '$lib/utils/asset-utils';
  import type { ContextMenuPosition } from '$lib/utils/context-menu';
  import { handleError } from '$lib/utils/handle-error';
  import { normalizeSearchString } from '$lib/utils/string-utils';
  import { addUsersToAlbum, deleteAlbum, isHttpError, type AlbumResponseDto, type AlbumUserAddDto } from '@immich/sdk';
  import { modalManager } from '@immich/ui';
  import { mdiDeleteOutline, mdiFolderDownloadOutline, mdiRenameOutline, mdiShareVariantOutline } from '@mdi/js';
  import { groupBy } from 'lodash-es';
  import { onMount, type Snippet } from 'svelte';
  import { t } from 'svelte-i18n';
  import { run } from 'svelte/legacy';

  interface Props {
    ownedAlbums?: AlbumResponseDto[];
    sharedAlbums?: AlbumResponseDto[];
    searchQuery?: string;
    userSettings: AlbumViewSettings;
    allowEdit?: boolean;
    showOwner?: boolean;
    albumGroupIds?: string[];
    empty?: Snippet;
  }

  let {
    ownedAlbums = $bindable([]),
    sharedAlbums = $bindable([]),
    searchQuery = '',
    userSettings,
    allowEdit = false,
    showOwner = false,
    albumGroupIds = $bindable([]),
    empty,
  }: Props = $props();

  interface AlbumGroupOption {
    [option: string]: (order: SortOrder, albums: AlbumResponseDto[]) => AlbumGroup[];
  }

  const groupOptions: AlbumGroupOption = {
    /** No grouping */
    [AlbumGroupBy.None]: (order, albums): AlbumGroup[] => {
      return [
        {
          id: $t('albums'),
          name: $t('albums'),
          albums,
        },
      ];
    },

    /** Group by year */
    [AlbumGroupBy.Year]: (order, albums): AlbumGroup[] => {
      const unknownYear = $t('unknown_year');
      const useStartDate = userSettings.sortBy === AlbumSortBy.OldestPhoto;

      const groupedByYear = groupBy(albums, (album) => {
        const date = useStartDate ? album.startDate : album.endDate;
        return date ? new Date(date).getFullYear() : unknownYear;
      });

      const sortSign = order === SortOrder.Desc ? -1 : 1;
      const sortedByYear = Object.entries(groupedByYear).sort(([a], [b]) => {
        // We make sure empty albums stay at the end of the list
        if (a === unknownYear) {
          return 1;
        } else if (b === unknownYear) {
          return -1;
        } else {
          return (Number.parseInt(a) - Number.parseInt(b)) * sortSign;
        }
      });

      return sortedByYear.map(([year, albums]) => ({
        id: year,
        name: year,
        albums,
      }));
    },

    /** Group by owner */
    [AlbumGroupBy.Owner]: (order, albums): AlbumGroup[] => {
      const currentUserId = $user.id;
      const groupedByOwnerIds = groupBy(albums, 'ownerId');

      const sortSign = order === SortOrder.Desc ? -1 : 1;
      const sortedByOwnerNames = Object.entries(groupedByOwnerIds).sort(([ownerA, albumsA], [ownerB, albumsB]) => {
        // We make sure owned albums stay either at the beginning or the end
        // of the list
        if (ownerA === currentUserId) {
          return -sortSign;
        } else if (ownerB === currentUserId) {
          return sortSign;
        } else {
          return albumsA[0].owner.name.localeCompare(albumsB[0].owner.name, $locale) * sortSign;
        }
      });

      return sortedByOwnerNames.map(([ownerId, albums]) => ({
        id: ownerId,
        name: ownerId === currentUserId ? $t('my_albums') : albums[0].owner.name,
        albums,
      }));
    },
  };

  let albums: AlbumResponseDto[] = $state([]);
  let filteredAlbums: AlbumResponseDto[] = $state([]);
  let groupedAlbums: AlbumGroup[] = $state([]);

  let albumGroupOption: string = $state(AlbumGroupBy.None);

  let albumToShare: AlbumResponseDto | null = $state(null);
  let albumToDelete: AlbumResponseDto | null = null;

  let contextMenuPosition: ContextMenuPosition = $state({ x: 0, y: 0 });
  let contextMenuTargetAlbum: AlbumResponseDto | undefined = $state();
  let isOpen = $state(false);

  // Step 1: Filter between Owned and Shared albums, or both.
  run(() => {
    switch (userSettings.filter) {
      case AlbumFilter.Owned: {
        albums = ownedAlbums;
        break;
      }
      case AlbumFilter.Shared: {
        albums = sharedAlbums;
        break;
      }
      default: {
        const userId = $user.id;
        const nonOwnedAlbums = sharedAlbums.filter((album) => album.ownerId !== userId);
        albums = nonOwnedAlbums.length > 0 ? ownedAlbums.concat(nonOwnedAlbums) : ownedAlbums;
      }
    }
  });

  // Step 2: Filter using the given search query.
  run(() => {
    if (searchQuery) {
      const searchAlbumNormalized = normalizeSearchString(searchQuery);

      filteredAlbums = albums.filter((album) => {
        return normalizeSearchString(album.albumName).includes(searchAlbumNormalized);
      });
    } else {
      filteredAlbums = albums;
    }
  });

  // Step 3: Group albums.
  run(() => {
    albumGroupOption = getSelectedAlbumGroupOption(userSettings);
    const groupFunc = groupOptions[albumGroupOption] ?? groupOptions[AlbumGroupBy.None];
    groupedAlbums = groupFunc(stringToSortOrder(userSettings.groupOrder), filteredAlbums);
  });

  // Step 4: Sort albums amongst each group.
  run(() => {
    groupedAlbums = groupedAlbums.map((group) => ({
      id: group.id,
      name: group.name,
      albums: sortAlbums(group.albums, { sortBy: userSettings.sortBy, orderBy: userSettings.sortOrder }),
    }));

    albumGroupIds = groupedAlbums.map(({ id }) => id);
  });

  let showFullContextMenu = $derived(
    allowEdit && contextMenuTargetAlbum && contextMenuTargetAlbum.ownerId === $user.id,
  );

  onMount(async () => {
    if (allowEdit) {
      await removeAlbumsIfEmpty();
    }
  });

  const showAlbumContextMenu = (contextMenuDetail: ContextMenuPosition, album: AlbumResponseDto) => {
    contextMenuTargetAlbum = album;
    contextMenuPosition = {
      x: contextMenuDetail.x,
      y: contextMenuDetail.y,
    };
    isOpen = true;
  };

  const closeAlbumContextMenu = () => {
    isOpen = false;
  };

  const handleDownloadAlbum = async () => {
    if (contextMenuTargetAlbum) {
      const album = contextMenuTargetAlbum;
      closeAlbumContextMenu();
      await downloadAlbum(album);
    }
  };

  const handleDeleteAlbum = async (albumToDelete: AlbumResponseDto) => {
    try {
      await deleteAlbum({
        id: albumToDelete.id,
      });
    } catch (error) {
      // In rare cases deleting an album completes after the list of albums has been requested,
      // leading to a bad request error.
      // Since the album is already deleted, the error is ignored.
      const isBadRequest = isHttpError(error) && error.status === 400;
      if (!isBadRequest) {
        throw error;
      }
    }

    ownedAlbums = ownedAlbums.filter(({ id }) => id !== albumToDelete.id);
    sharedAlbums = sharedAlbums.filter(({ id }) => id !== albumToDelete.id);
  };

  const setAlbumToDelete = async () => {
    albumToDelete = contextMenuTargetAlbum ?? null;
    closeAlbumContextMenu();
    await deleteSelectedAlbum();
  };

  const handleEdit = async (album: AlbumResponseDto) => {
    closeAlbumContextMenu();
    const editedAlbum = await modalManager.show(AlbumEditModal, {
      album,
    });
    if (editedAlbum) {
      successEditAlbumInfo(editedAlbum);
    }
  };

  const deleteSelectedAlbum = async () => {
    if (!albumToDelete) {
      return;
    }

    const isConfirmed = await confirmAlbumDelete(albumToDelete);

    if (!isConfirmed) {
      return;
    }

    try {
      await handleDeleteAlbum(albumToDelete);
    } catch {
      notificationController.show({
        message: $t('errors.unable_to_delete_album'),
        type: NotificationType.Error,
      });
    } finally {
      albumToDelete = null;
    }
  };

  const removeAlbumsIfEmpty = async () => {
    const albumsToRemove = ownedAlbums.filter((album) => album.assetCount === 0 && !album.albumName);
    await Promise.allSettled(albumsToRemove.map((album) => handleDeleteAlbum(album)));
  };

  const updateAlbumInfo = (album: AlbumResponseDto) => {
    ownedAlbums[ownedAlbums.findIndex(({ id }) => id === album.id)] = album;
    sharedAlbums[sharedAlbums.findIndex(({ id }) => id === album.id)] = album;
  };

  const updateRecentAlbumInfo = (album: AlbumResponseDto) => {
    for (const cachedAlbum of userInteraction.recentAlbums || []) {
      if (cachedAlbum.id === album.id) {
        Object.assign(cachedAlbum, { ...cachedAlbum, ...album });
        break;
      }
    }
  };

  const successEditAlbumInfo = (album: AlbumResponseDto) => {
    notificationController.show({
      message: $t('album_info_updated'),
      type: NotificationType.Info,
      button: {
        text: $t('view_album'),
        onClick() {
          return goto(`${AppRoute.ALBUMS}/${album.id}`);
        },
      },
    });

    updateAlbumInfo(album);
    updateRecentAlbumInfo(album);
  };

  const handleAddUsers = async (albumUsers: AlbumUserAddDto[]) => {
    if (!albumToShare) {
      return;
    }
    try {
      const album = await addUsersToAlbum({
        id: albumToShare.id,
        addUsersDto: {
          albumUsers,
        },
      });
      updateAlbumInfo(album);
    } catch (error) {
      handleError(error, $t('errors.unable_to_add_album_users'));
    } finally {
      albumToShare = null;
    }
  };

  const handleSharedLinkCreated = (album: AlbumResponseDto) => {
    album.shared = true;
    album.hasSharedLink = true;
    updateAlbumInfo(album);
  };

  const openShareModal = async () => {
    if (!contextMenuTargetAlbum) {
      return;
    }

    albumToShare = contextMenuTargetAlbum;
    closeAlbumContextMenu();
    const result = await modalManager.show(AlbumShareModal, { album: albumToShare });

    switch (result?.action) {
      case 'sharedUsers': {
        await handleAddUsers(result.data);
        return;
      }

      case 'sharedLink': {
        const sharedLink = await modalManager.show(SharedLinkCreateModal, { albumId: albumToShare.id });

        if (sharedLink) {
          handleSharedLinkCreated(albumToShare);
          await modalManager.show(QrCodeModal, { title: $t('view_link'), value: makeSharedLinkUrl(sharedLink) });
        }
        return;
      }
    }
  };
</script>

{#if albums.length > 0}
  {#if userSettings.view === AlbumViewMode.Cover}
    <!-- Album Cards -->
    {#if albumGroupOption === AlbumGroupBy.None}
      <AlbumCardGroup
        albums={groupedAlbums[0].albums}
        {showOwner}
        showDateRange
        showItemCount
        onShowContextMenu={showAlbumContextMenu}
      />
    {:else}
      {#each groupedAlbums as albumGroup (albumGroup.id)}
        <AlbumCardGroup
          albums={albumGroup.albums}
          group={albumGroup}
          {showOwner}
          showDateRange
          showItemCount
          onShowContextMenu={showAlbumContextMenu}
        />
      {/each}
    {/if}
  {:else if userSettings.view === AlbumViewMode.List}
    <!-- Album Table -->
    <AlbumsTable {groupedAlbums} {albumGroupOption} onShowContextMenu={showAlbumContextMenu} />
  {/if}
{:else}
  <!-- Empty Message -->
  {@render empty?.()}
{/if}

<!-- Context Menu -->
<RightClickContextMenu title={$t('album_options')} {...contextMenuPosition} {isOpen} onClose={closeAlbumContextMenu}>
  {#if showFullContextMenu}
    <MenuOption
      icon={mdiRenameOutline}
      text={$t('edit_album')}
      onClick={() => contextMenuTargetAlbum && handleEdit(contextMenuTargetAlbum)}
    />
    <MenuOption icon={mdiShareVariantOutline} text={$t('share')} onClick={() => openShareModal()} />
  {/if}
  <MenuOption icon={mdiFolderDownloadOutline} text={$t('download')} onClick={() => handleDownloadAlbum()} />
  {#if showFullContextMenu}
    <MenuOption icon={mdiDeleteOutline} text={$t('delete')} onClick={() => setAlbumToDelete()} />
  {/if}
</RightClickContextMenu>
