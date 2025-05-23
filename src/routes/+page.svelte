<script lang="ts">
  //import { invoke } from "@tauri-apps/api/core";
  import { onMount } from "svelte";
  import * as fs from "@tauri-apps/plugin-fs";
  import * as path from "@tauri-apps/api/path";
  import { BaseDirectory } from "@tauri-apps/api/path";
  import { open, save } from "@tauri-apps/plugin-dialog";

  const isDevelopment = import.meta.env.DEV;

  interface SetupDetail {
    Category: string;
    Details: string;
    BulletPoints: string[];
  }

  interface CommonIssue {
    Question: string;
    Answer: string;
  }

  const MultiplayerType = {
    None: 0,
    Local: 1,
    LAN: 2,
    Online: 3,
    Unknown: 4,
  } as const;

  const CompatibilityStatus = {
    Perfect: 0,
    Great: 1,
    Issues: 2,
    Unplayable: 3,
    Unknown: 4,
  } as const;

  type MultiplayerTypeValue = (typeof MultiplayerType)[keyof typeof MultiplayerType];
  type CompatibilityStatusValue = (typeof CompatibilityStatus)[keyof typeof CompatibilityStatus];

  interface GameEntry {
    Id: string;
    Name: string;
    Hardware: string;
    Description: string;
    IconUrl: string;
    Genre: string;
    MultiplayerType: MultiplayerTypeValue;
    MultiplayerDetails: string | null;
    NvidiaSupport: CompatibilityStatusValue;
    AmdSupport: CompatibilityStatusValue;
    IntelSupport: CompatibilityStatusValue;
    SetupDetails: SetupDetail[];
    CommonIssues: CommonIssue[];
    FeaturesNotEmulated: string[];
    OverallStatus: CompatibilityStatusValue;
    [key: string]: any;
  }

  const statusMap = {
    [CompatibilityStatus.Perfect]: "Perfect",
    [CompatibilityStatus.Great]: "Great",
    [CompatibilityStatus.Issues]: "Issues",
    [CompatibilityStatus.Unplayable]: "Unplayable",
    [CompatibilityStatus.Unknown]: "Unknown",
  };

  const multiplayerTypeMap = {
    [MultiplayerType.None]: "None",
    [MultiplayerType.Local]: "Local",
    [MultiplayerType.LAN]: "LAN",
    [MultiplayerType.Online]: "Online",
    [MultiplayerType.Unknown]: "Unknown",
  };
  let items = $state<GameEntry[]>([]);
  let selectedItem = $state<number | null>(null);

  async function loadJsonData() {
    try {
      let jsonContent;

      if (isDevelopment) {
        // Development mode - try to load from known locations
        try {
          jsonContent = await fs.readTextFile("gameCompatibility.json");
        } catch {
          try {
            jsonContent = await fetch("/gameCompatibility.json").then((r) => r.text());
          } catch {
            const home = await path.homeDir();
            const jsonPath = await path.join(home, "gameCompatibility.json");
            jsonContent = await fs.readTextFile(jsonPath);
          }
        }
      } else {
        // Production mode - use file picker
        const selected = await open({
          title: "Load Game Compatibility Data",
          filters: [
            {
              name: "JSON Files",
              extensions: ["json"],
            },
          ],
        });

        if (!selected) return;
        jsonContent = await fs.readTextFile(selected as string);
      }

      const parsedItems = JSON.parse(jsonContent) as GameEntry[];
      items = parsedItems.sort((a, b) => a.Name.localeCompare(b.Name));
    } catch (error) {
      console.error("Failed to load JSON data:", error);
    }
  }

  async function saveJsonData() {
    try {
      const jsonContent = JSON.stringify(items, null, 2);
      const path = await save({
        title: "Save Game Compatibility Data",
        defaultPath: "gameCompatibility.json",
        filters: [
          {
            name: "gameCompatibility.json",
            extensions: ["json"],
          },
        ],
      });
      if (!path) {
        return;
      }
      await fs.writeTextFile(path, jsonContent, {
        baseDir: BaseDirectory.AppConfig,
      });
      alert("Data saved successfully");
    } catch (error) {
      console.error("Failed to save JSON data:", error);
      alert("Failed to save data");
    }
  }

  function selectItem(index: number) {
    selectedItem = index;
  }

  function handleKeyDown(event: KeyboardEvent, index: number) {
    if (event.key === "Enter" || event.key === " ") {
      event.preventDefault();
      selectItem(index);
    }
  }

  function updateField(field: keyof GameEntry, value: any) {
    if (selectedItem !== null) {
      items[selectedItem][field] = value;
    }
  }

  function handleSelectChange(e: Event, field: string) {
    const target = e.target as HTMLSelectElement;
    if (target && target.value) {
      updateField(field, parseInt(target.value));
    }
  }

  function addArrayItem(field: keyof GameEntry, template: any = "") {
    if (selectedItem !== null) {
      const currentArray = [...items[selectedItem][field]];
      currentArray.push(template);
      updateField(field, currentArray);
    }
  }

  function removeArrayItem(field: keyof GameEntry, index: number) {
    if (selectedItem !== null) {
      const currentArray = [...items[selectedItem][field]];
      currentArray.splice(index, 1);
      updateField(field, currentArray);
    }
  }

  function updateArrayItem(field: keyof GameEntry, index: number, value: any) {
    if (selectedItem !== null) {
      const currentArray = [...items[selectedItem][field]];
      currentArray[index] = value;
      updateField(field, currentArray);
    }
  }

  function updateBulletPoint(setupIndex: number, bulletIndex: number, value: string) {
    if (selectedItem !== null) {
      const setupDetails = [...items[selectedItem].SetupDetails];
      const bulletPoints = [...setupDetails[setupIndex].BulletPoints];
      bulletPoints[bulletIndex] = value;
      setupDetails[setupIndex].BulletPoints = bulletPoints;
      updateField("SetupDetails", setupDetails);
    }
  }

  function addBulletPoint(setupIndex: number) {
    if (selectedItem !== null) {
      const setupDetails = [...items[selectedItem].SetupDetails];
      setupDetails[setupIndex].BulletPoints.push("");
      updateField("SetupDetails", setupDetails);
    }
  }

  function removeBulletPoint(setupIndex: number, bulletIndex: number) {
    if (selectedItem !== null) {
      const setupDetails = [...items[selectedItem].SetupDetails];
      setupDetails[setupIndex].BulletPoints.splice(bulletIndex, 1);
      updateField("SetupDetails", setupDetails);
    }
  }

  onMount(() => {
    loadJsonData();
  });
</script>

<main class="flex flex-col h-screen">
  <div class="navbar bg-base-100 shadow-sm">
    <div class="flex-1">
      <p class="btn btn-ghost text-xl">TP Compatibility Editor</p>
    </div>
    <div class="flex-none">
      <button class="btn btn-soft btn-info" onclick={loadJsonData}> Load Data </button>
      <button class="btn btn-soft btn-success" onclick={saveJsonData}> Save Data </button>
    </div>
  </div>

  <div class="flex flex-1 overflow-hidden p-4">
    <div class="w-1/4 border-r border-base-300 pr-4">
      <div class="h-full overflow-y-auto">
        <ul class="space-y-1" role="listbox" aria-label="Items list">
          {#each items as item, index}
            <button
              type="button"
              role="option"
              aria-selected={selectedItem === index}
              class="p-2 cursor-pointer rounded text-left w-full hover:bg-gray-100 {selectedItem ===
              index
                ? 'bg-blue-100'
                : ''}"
              onclick={() => selectItem(index)}
              onkeydown={(e) => handleKeyDown(e, index)}
            >
              {item.Name}
            </button>
          {/each}
        </ul>
      </div>
    </div>

    <div class="w-3/4 pl-4 overflow-y-auto">
      {#if selectedItem !== null}
        <div class="space-y-4">
          <h2 class="text-xl font-semibold">{items[selectedItem].Name}</h2>

          <div class="grid grid-cols-2 gap-4">
            <div class="form-control">
              <label class="label" for="game-id">
                <span class="label-text">Game ID</span>
              </label>
              <input
                id="game-id"
                type="text"
                class="input input-bordered w-full focus:outline-none"
                value={items[selectedItem].Id}
                onchange={(e) => updateField("Id", (e.target as HTMLInputElement).value)}
              />
            </div>

            <div class="form-control">
              <label class="label" for="game-name">
                <span class="label-text">Name</span>
              </label>
              <input
                id="game-name"
                type="text"
                class="input input-bordered w-full focus:outline-none"
                value={items[selectedItem].Name}
                onchange={(e) => updateField("Name", (e.target as HTMLInputElement).value)}
              />
            </div>

            <div class="form-control">
              <label class="label" for="hardware">
                <span class="label-text">Hardware</span>
              </label>
              <input
                id="hardware"
                type="text"
                class="input input-bordered w-full focus:outline-none"
                value={items[selectedItem].Hardware}
                onchange={(e) => updateField("Hardware", (e.target as HTMLInputElement).value)}
              />
            </div>

            <div class="form-control">
              <label class="label" for="genre">
                <span class="label-text">Genre</span>
              </label>
              <input
                id="genre"
                type="text"
                class="input input-bordered w-full focus:outline-none"
                value={items[selectedItem].Genre}
                onchange={(e) => updateField("Genre", (e.target as HTMLInputElement).value)}
              />
            </div>

            <div class="form-control">
              <label class="label" for="multiplayer-type">
                <span class="label-text">Multiplayer Type</span>
              </label>
              <select
                id="multiplayer-type"
                class="select select-bordered w-full focus:outline-none"
                value={items[selectedItem].MultiplayerType.toString()}
                onchange={(e) => handleSelectChange(e, "MultiplayerType")}
              >
                {#each Object.entries(multiplayerTypeMap) as [value, label]}
                  <option {value}>{label}</option>
                {/each}
              </select>
            </div>

            <div class="form-control">
              <label class="label" for="overall-status">
                <span class="label-text">Overall Status</span>
              </label>
              <select
                id="overall-status"
                class="select select-bordered w-full focus:outline-none"
                value={items[selectedItem].OverallStatus.toString()}
                onchange={(e) => handleSelectChange(e, "OverallStatus")}
              >
                {#each Object.entries(statusMap) as [value, label]}
                  <option {value}>{label}</option>
                {/each}
              </select>
            </div>

            <div class="form-control">
              <label class="label" for="nvidia-support">
                <span class="label-text">NVIDIA Support</span>
              </label>
              <select
                id="nvidia-support"
                class="select select-bordered w-full focus:outline-none"
                value={items[selectedItem].NvidiaSupport.toString()}
                onchange={(e) => handleSelectChange(e, "NvidiaSupport")}
              >
                {#each Object.entries(statusMap) as [value, label]}
                  <option {value}>{label}</option>
                {/each}
              </select>
            </div>

            <div class="form-control">
              <label class="label" for="amd-support">
                <span class="label-text">AMD Support</span>
              </label>
              <select
                id="amd-support"
                class="select select-bordered w-full focus:outline-none"
                value={items[selectedItem].AmdSupport.toString()}
                onchange={(e) => handleSelectChange(e, "AmdSupport")}
              >
                {#each Object.entries(statusMap) as [value, label]}
                  <option {value}>{label}</option>
                {/each}
              </select>
            </div>

            <div class="form-control">
              <label class="label" for="intel-support">
                <span class="label-text">Intel Support</span>
              </label>
              <select
                id="intel-support"
                class="select select-bordered w-full focus:outline-none"
                value={items[selectedItem].IntelSupport.toString()}
                onchange={(e) => handleSelectChange(e, "IntelSupport")}
              >
                {#each Object.entries(statusMap) as [value, label]}
                  <option {value}>{label}</option>
                {/each}
              </select>
            </div>
          </div>

          <div class="form-control">
            <label class="label" for="description">
              <span class="label-text">Description</span>
            </label>
            <textarea
              id="description"
              class="textarea textarea-bordered w-full h-24 focus:outline-none"
              value={items[selectedItem].Description}
              onchange={(e) => updateField("Description", (e.target as HTMLTextAreaElement).value)}
            ></textarea>
          </div>

          <div class="form-control">
            <label class="label" for="icon-url">
              <span class="label-text">Icon URL</span>
            </label>
            <input
              id="icon-url"
              type="text"
              class="input input-bordered w-full focus:outline-none"
              value={items[selectedItem].IconUrl}
              onchange={(e: Event) => {
                const target = e.target as HTMLInputElement;
                if (target) updateField("IconUrl", target.value);
              }}
            />
          </div>

          <div class="form-control">
            <label class="label" for="multiplayer-details">
              <span class="label-text">Multiplayer Details</span>
            </label>
            <input
              id="multiplayer-details"
              type="text"
              class="input input-bordered w-full focus:outline-none"
              value={items[selectedItem].MultiplayerDetails || ""}
              onchange={(e) =>
                updateField("MultiplayerDetails", (e.target as HTMLInputElement).value)}
            />
          </div>

          <!-- Setup Details Section -->
          <div class="form-control">
            <label class="label">
              <span class="label-text font-bold">Setup Details</span>
              <button
                class="btn btn-sm btn-outline"
                onclick={() =>
                  addArrayItem("SetupDetails", { Category: "", Details: "", BulletPoints: [] })}
              >
                Add Setup Detail
              </button>
            </label>

            {#if items[selectedItem].SetupDetails.length === 0}
              <p class="text-gray-500 italic">No setup details added</p>
            {:else}
              {#each items[selectedItem].SetupDetails as setup, setupIndex}
                <div class="card bg-base-200 shadow-sm p-4 mb-4">
                  <div class="flex justify-between items-center mb-2">
                    <h3 class="font-semibold">Setup Detail #{setupIndex + 1}</h3>
                    <button
                      class="btn btn-sm btn-error"
                      onclick={() => removeArrayItem("SetupDetails", setupIndex)}
                    >
                      Remove
                    </button>
                  </div>

                  <div class="form-control mb-2">
                    <label class="label" for={`setup-category-${setupIndex}`}>
                      <span class="label-text">Category</span>
                    </label>
                    <input
                      id={`setup-category-${setupIndex}`}
                      type="text"
                      class="input input-bordered w-full focus:outline-none"
                      value={setup.Category}
                      onchange={(e) => {
                        const updatedSetup = {
                          ...setup,
                          Category: (e.target as HTMLInputElement).value,
                        };
                        updateArrayItem("SetupDetails", setupIndex, updatedSetup);
                      }}
                    />
                  </div>

                  <div class="form-control mb-2">
                    <label class="label" for={`setup-details-${setupIndex}`}>
                      <span class="label-text">Details</span>
                    </label>
                    <textarea
                      id={`setup-details-${setupIndex}`}
                      class="textarea textarea-bordered w-full h-20 focus:outline-none"
                      value={setup.Details}
                      onchange={(e) => {
                        const updatedSetup = {
                          ...setup,
                          Details: (e.target as HTMLTextAreaElement).value,
                        };
                        updateArrayItem("SetupDetails", setupIndex, updatedSetup);
                      }}
                    ></textarea>
                  </div>

                  <div class="form-control">
                    <label class="label">
                      <span class="label-text">Bullet Points</span>
                      <button
                        class="btn btn-sm btn-outline"
                        onclick={() => addBulletPoint(setupIndex)}
                      >
                        Add Point
                      </button>
                    </label>

                    {#if setup.BulletPoints.length === 0}
                      <p class="text-gray-500 italic">No bullet points added</p>
                    {:else}
                      {#each setup.BulletPoints as point, bulletIndex}
                        <div class="flex gap-2 mb-2">
                          <input
                            type="text"
                            class="input input-bordered w-full focus:outline-none"
                            value={point}
                            onchange={(e) =>
                              updateBulletPoint(
                                setupIndex,
                                bulletIndex,
                                (e.target as HTMLInputElement).value
                              )}
                          />
                          <button
                            class="btn btn-sm btn-error"
                            onclick={() => removeBulletPoint(setupIndex, bulletIndex)}
                          >
                            -
                          </button>
                        </div>
                      {/each}
                    {/if}
                  </div>
                </div>
              {/each}
            {/if}
          </div>

          <!-- Common Issues Section -->
          <div class="form-control">
            <label class="label">
              <span class="label-text font-bold">Common Issues</span>
              <button
                class="btn btn-sm btn-outline"
                onclick={() => addArrayItem("CommonIssues", { Question: "", Answer: "" })}
              >
                Add Issue
              </button>
            </label>

            {#if items[selectedItem].CommonIssues.length === 0}
              <p class="text-gray-500 italic">No common issues added</p>
            {:else}
              {#each items[selectedItem].CommonIssues as issue, issueIndex}
                <div class="card bg-base-200 shadow-sm p-4 mb-4">
                  <div class="flex justify-between items-center mb-2">
                    <h3 class="font-semibold">Issue #{issueIndex + 1}</h3>
                    <button
                      class="btn btn-sm btn-error"
                      onclick={() => removeArrayItem("CommonIssues", issueIndex)}
                    >
                      Remove
                    </button>
                  </div>

                  <div class="form-control mb-2">
                    <label class="label" for="issue-question">
                      <span class="label-text">Question</span>
                    </label>
                    <input
                      id="issue-question"
                      type="text"
                      class="input input-bordered w-full focus:outline-none"
                      value={issue.Question}
                      onchange={(e) => {
                        const updatedIssue = {
                          ...issue,
                          Question: (e.target as HTMLInputElement).value,
                        };
                        updateArrayItem("CommonIssues", issueIndex, updatedIssue);
                      }}
                    />
                  </div>

                  <div class="form-control">
                    <label class="label" for="issue-answer">
                      <span class="label-text">Answer</span>
                    </label>
                    <textarea
                      id="issue-answer"
                      class="textarea textarea-bordered w-full h-20 focus:outline-none"
                      value={issue.Answer}
                      onchange={(e) => {
                        const updatedIssue = {
                          ...issue,
                          Answer: (e.target as HTMLTextAreaElement).value,
                        };
                        updateArrayItem("CommonIssues", issueIndex, updatedIssue);
                      }}
                    ></textarea>
                  </div>
                </div>
              {/each}
            {/if}
          </div>

          <!-- Features Not Emulated Section -->
          <div class="form-control">
            <label class="label">
              <span class="label-text font-bold">Features Not Emulated</span>
              <button
                class="btn btn-sm btn-outline"
                onclick={() => addArrayItem("FeaturesNotEmulated", "")}
              >
                Add Feature
              </button>
            </label>

            {#if items[selectedItem].FeaturesNotEmulated.length === 0}
              <p class="text-gray-500 italic">No features listed</p>
            {:else}
              {#each items[selectedItem].FeaturesNotEmulated as feature, featureIndex}
                <div class="flex gap-2 mb-2">
                  <input
                    type="text"
                    class="input input-bordered w-full focus:outline-none"
                    value={feature}
                    onchange={(e) => {
                      if (selectedItem !== null) {
                        const updatedFeatures = [...items[selectedItem].FeaturesNotEmulated];
                        updatedFeatures[featureIndex] = (e.target as HTMLInputElement).value;
                        updateField("FeaturesNotEmulated", updatedFeatures);
                      }
                    }}
                  />
                  <button
                    class="btn btn-sm btn-error"
                    onclick={() => removeArrayItem("FeaturesNotEmulated", featureIndex)}
                  >
                    Remove
                  </button>
                </div>
              {/each}
            {/if}
          </div>
        </div>
      {:else}
        <div class="">Select an item from the list</div>
      {/if}
    </div>
  </div>
</main>

<style>
  ul {
    list-style-type: none;
    padding: 0;
  }

  .form-control {
    margin-bottom: 0.5rem;
  }

  .label {
    padding: 0.25rem 0;
  }

  .label-text {
    font-weight: 500;
  }
</style>
