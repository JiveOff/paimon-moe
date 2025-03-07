<script>
  import { t } from 'svelte-i18n';

  import { onMount } from 'svelte';
  import dayjs from 'dayjs';

  import { characters } from '../../data/characters';
  import { weaponList } from '../../data/weaponList';
  import { bannerTypes } from '../../data/bannerTypes';

  import { getAccountPrefix } from '../../stores/account';
  import { readSave, updateTime, fromRemote, updateSave } from '../../stores/saveManager';
  import SummaryItem from './_summaryItem.svelte';
  import Icon from '../../components/Icon.svelte';
  import { mdiEarth } from '@mdi/js';

  let numberFormat = Intl.NumberFormat();

  export let monthlyData = {};

  const types = bannerTypes;

  let loading = true;
  let wishCount = 0;
  const avg = {};

  $: if ($fromRemote) {
    readLocalData();
  }

  $: if ($updateTime) {
    setTimeout(() => {
      readLocalData();
    }, 1000);
  }

  onMount(async () => {
    await readLocalData();
  });

  const defaultChars = {
    amber: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    kaeya: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    lisa: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    traveler_geo: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    traveler_anemo: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    barbara: {
      default: 1,
      wish: 0,
      manual: 0,
    },
    xiangling: {
      default: 1,
      wish: 0,
      manual: 0,
    },
  };

  export async function readLocalData() {
    let totalWish = 0;
    console.log('wish summary read local');
    const prefix = getAccountPrefix();

    let currentMonthlyData = {};

    // collected characters stuff
    let updateCollectedCharacters = false;
    let collectedCharacters = {};
    const collectedCharactersData = await readSave(`${prefix}characters`);
    if (collectedCharactersData !== null) {
      collectedCharacters = collectedCharactersData;
      for (const collectedId of Object.keys(collectedCharacters)) {
        collectedCharacters[collectedId].wish = 0;
      }
      updateCollectedCharacters = true;
    } else {
      collectedCharacters = {...defaultChars};
    }
    const collectablesNeedUpdateData = await readSave(`${prefix}collectables-updated`);
    if (collectablesNeedUpdateData === null || collectablesNeedUpdateData === true) {
      updateCollectedCharacters = true;
    } else if (collectablesNeedUpdateData === false) {
      updateCollectedCharacters = false;
    }

    for (let type of types) {
      const path = `wish-counter-${type.id}`;
      const data = await readSave(`${prefix}${path}`);
      if (data !== null) {
        const counterData = data;
        const pulls = counterData.pulls || [];
        const total = counterData.total;

        totalWish += total;

        let legendary = 0;
        let legendaryPity = 0;
        let legendaryPulls = [];
        let rare = 0;
        let rareWeapon = 0;
        let rarePityWeapon = 0;
        let rareCharacter = 0;
        let rarePityCharacter = 0;
        let rarePity = 0;
        for (let pull of pulls) {
          let rarity;
          let itemName;
          let currentType;
          if (pull.type === 'character') {
            rarity = characters[pull.id].rarity;
            itemName = characters[pull.id].name;
            currentType = 'character';

            // collected characters stuff
            if (updateCollectedCharacters) {
              if (collectedCharacters[pull.id]) {
                collectedCharacters[pull.id].wish += 1;
              } else {
                collectedCharacters[pull.id] = {
                  default: 0,
                  manual: 0,
                  wish: 1,
                };
              }
            }
          } else if (pull.type === 'weapon') {
            rarity = weaponList[pull.id].rarity;
            itemName = weaponList[pull.id].name;
            currentType = 'weapon';
          }

          const time = dayjs(pull.time).format('YYYY-MM');
          if (currentMonthlyData[time] === undefined) {
            currentMonthlyData[time] = {
              total: 0,
              legendary: 0,
              rare: 0,
            };
          }

          currentMonthlyData[time].total++;

          if (rarity === 5) {
            legendary++;
            legendaryPity += pull.pity;
            currentMonthlyData[time].legendary++;

            legendaryPulls.push({ name: itemName, pity: pull.pity });
          } else if (rarity === 4) {
            rare++;
            rarePity += pull.pity;
            currentMonthlyData[time].rare++;

            if (currentType === 'character') {
              rareCharacter++;
              rarePityCharacter += pull.pity;
            } else if (currentType === 'weapon') {
              rareWeapon++;
              rarePityWeapon += pull.pity;
            }
          }
        }

        avg[type.id] = {
          rare: {
            total: rare,
            percentage: total > 0 ? rare / total : 0,
            pity: rare > 0 ? rarePity / rare : 0,
            weapon: {
              total: rareWeapon,
              percentage: total > 0 ? rareWeapon / total : 0,
              pity: rare > 0 ? rarePityWeapon / rare : 0,
            },
            character: {
              total: rareCharacter,
              percentage: total > 0 ? rareCharacter / total : 0,
              pity: rare > 0 ? rarePityCharacter / rare : 0,
            },
          },
          legendary: {
            total: legendary,
            percentage: total > 0 ? legendary / total : 0,
            pity: legendary > 0 ? legendaryPity / legendary : 0,
            pulls: legendaryPulls,
          },
        };
      }
    }

    wishCount = totalWish;
    monthlyData = currentMonthlyData;

    if (updateCollectedCharacters && totalWish > 0) {
      console.log('updating collectables');
      await updateSave(`${prefix}characters`, collectedCharacters);
      await updateSave(`${prefix}collectables-updated`, false);
    }

    // console.log(avg);
    loading = false;
  }

</script>

{#if !loading}
  <div class="flex flex-col space-y-4">
    {#if avg[types[0].id]}
      <SummaryItem avg={avg[types[0].id]} type={types[0]} />
    {/if}
    {#if avg[types[1].id]}
      <SummaryItem avg={avg[types[1].id]} type={types[1]} />
    {/if}
  </div>
  <div class="flex flex-col space-y-4">
    {#if avg[types[2].id]}
      <SummaryItem avg={avg[types[2].id]} type={types[2]} />
    {/if}
    {#if avg[types[3].id]}
      <SummaryItem avg={avg[types[3].id]} type={types[3]} />
    {/if}
    <div class="bg-item rounded-xl p-4 flex items-center w-full text-white" style="height: min-content;">
      {$t('wish.wishesWorth')} <img class="w-4 h-4 mx-2" src="/images/primogem.png" alt="primogem" />
      {numberFormat.format(wishCount * 160)}
    </div>
    <a
      href="/wish/tally"
      class="bg-item rounded-xl p-4 flex items-center w-full text-white hover:text-primary"
      style="height: min-content;"
    >
      <Icon path={mdiEarth} className="mr-2" />
      {$t('wish.globalWishTally')}
    </a>
  </div>
{/if}
