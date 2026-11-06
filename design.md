# Current Year: design notes

A single-file first-person game. It runs on three.js r128 (UMD build from cdnjs). Characters are baked offline from the CC0 MakeHuman base mesh and embedded in the page as gzip+base64. The page unpacks them at runtime with `DecompressionStream`.

## Premise and flow
1. You start on the walk in front of your parents' house on Pine Street.
   - Mom tells you you're 24, need a job, and can't live there anymore.
   - She gives you $20.
   - Knocking again only gets "did you find a job yet?".
2. Get around on the 14 Crosstown. Every stop lists every other location. The fare is $1.50, and each paid ride gives you a transfer for the next ride.
3. The four locations:
   - **Pine Street:** the house, a vending machine and a trash can.
   - **Fresh Basket:** grocery store with Sharon on the register, plus a can return machine, the parking-lot cans and shoppers Dana and Dale.
   - **Ridgeway Outfitters:** clothing store with Tyler. It has racks, a hat table, a boot wall, mannequins and a working mirror.
   - **ValuMart:** big-box store with Kayla. It sells tents, sleeping bags, pens and cheap food, and has a parking lot and dumpsters out back.

## Systems
- **Time.** The clock follows your device's local time, plus any hours you've slept (`timeOffset`).
  - Sky, sun position, lighting and street lights are blended between night, dusk and day keyframes.
- **Needs.** Food drains to empty in about 2 hours, water in about 1.5 hours and energy in 16 hours of play. Running drains them faster.
  - Empty meters slow you down and stop you running.
  - Eating and drinking happen from the bag. Sodas give +7 energy and coffee +14. Every soda leaves an empty can you can return.
- **Reputation.** It drops when someone sees you:
  - picking up cans (−1)
  - digging in trash (−4 to −5)
  - pitching or sleeping in a tent (−8 to −10)

  It slowly recovers over time.
- **Line of sight.** `witnesses()` ray-tests from every NPC and clerk in the current location against the collision boxes. `passerby()` adds a chance based on how close you are to busy spots (`TRAFFIC`), with walls cutting that chance.
- **Camping.**
  - **Pitching:** pitch from the bag, outdoors, off the road. If anyone sees you, the deputy writes a $25 citation and you pack up.
  - **Sleeping:** choose 1–10 hours. Each hour rolls the tent spot's exposure. If you get caught, you wake up with a ticket and keep only the energy from hours actually slept.
  - **Recovery:** about 10 energy per hour slept (13 with a sleeping bag), with a bonus for longer sleeps.
  - **Fines:** can push your cash negative.
- **Jobs.** Ask any clerk for an application, and buy a pen at ValuMart.
  - **Filling it out:** a mini-game. You answer five fields, then scribble a signature until the meter fills.
  - **Turning it in:** "We aren't really hiring, but we'll call you"; after that, "We'll get back to you."
  - **The email:** 2–3 in-game days later a corporate HR rejection arrives on the Phone (press P or tap Phone).
- **Inventory and equipment.**
  - **Bag:** 16 slots. Stack sizes depend on the kind of item.
  - **Wear tab:** has a rotating 3D preview of your character and six slots: hat, eyewear, top, outer, bottoms and feet.
  - **Shop:** 29 clothing items. Ridgeway carries most of them and ValuMart the cheap ones.
- **Saving.** Progress is stored in localStorage (`currentyear-v1`).

## Characters (shoppers/)
- `bake3.py` builds each person with these helpers:
  - `body.py`: MakeHuman macro targets (age, weight, muscle) plus face targets, vertex-colour skin detail and a hairline.
  - `garments.py`: clothing pieces fitted as tubes around each limb.
    - Sleeves follow the arm at the 85th-percentile radius. Pant legs run straight from the knee.
    - Edges are cut cleanly, with hems turned under.
    - Also covers the apron, shoes built from shape parameters, and a cap and beanie shaped to the head.
  - `hair.py`: strand cards for buns and bobs.
- Short hair and eyebrows are fur shells generated at runtime from the scalp and brow vertices.
- Every garment kind is baked once per body. The game shows or hides garments and drops the body triangles they cover, using a per-vertex cover bitmask.
- `garmentMat` draws fabric detail procedurally and anti-aliased: knit, denim twill with fades and stitching, flannel plaid, ribbed cuffs, a polo placket, a fleece pocket, a jacket zip, shoe laces and soles.
- Walking is CMU motion capture retargeted onto each character (`mocap.py`).
- The cast is rural Oregon locals: Dana, Dale, Sharon, Tyler, Kayla and you.

## Round 3 additions
- **Store hours.** Fresh Basket is open 6am–10pm, ValuMart 7am–11pm and Ridgeway 9am–7pm (`HOURS`).
  - Every store has automatic sliding doors (`autoDoor`). They lock when the store is closed.
  - When a store closes, its lights dim and its clerk and shoppers leave. If you're inside at closing, you get walked out.
- **Parking lots.**
  - Ridgeway has a side lot, and ValuMart has a two-row lot with a crosswalk.
  - Each lot gets new cans once per game day (`regenCans`), up to 18 per lot.
  - Security guards patrol the lots from 9pm to 6am. They count as witnesses.
- **ValuMart interior.** Six aisles stocked with instanced products, plus hanging aisle signs, end caps, a back wall of coolers, four checkouts, carts, water pallets, apparel racks and a camping section.
- **Trash.** Each can and dumpster can be searched once per game day. More of them sit behind every store.
- **Rats.** They spawn daily at each trash can (two at a dumpster). Killing one gives 12 XP and rat meat, which has a 45% chance to make you sick.
- **Combat.**
  - Weapons: bat, knife, crowbar and hatchet, sold at ValuMart and Ridgeway. Equip one in the Hand slot, then press F or tap Hit.
  - You can attack anyone. Each person either fights back or runs (`FIGHTERS` sets the odds), and knocked-out people leave.
  - If they run, get knocked out, or anyone sees it, the cops are called. You're arrested 10–16 seconds later.
- **Jail.** An arrest means 24 hours in jail. Bail is the total of your charges: assault $250, car break-in $400, attempted break-in $150. If you can't pay, it's game over and the save is wiped.
- **Cars.**
  - Cars are now sedans and pickups with rounded bodies, wheel arches, rims, glass, mirrors, bumpers, plates and lights.
  - You can break into any parked car. An alarm brings security or a neighbor, who report it. A crowbar and Lockpicking skill lower the alarm chance.
  - A successful break-in can turn up cash or items.
- **Skills.** Fighting, Scavenging, Stealth, Lockpicking, Survival and Charm. Each improves as you use it, up to level 10, and makes its activity easier.
- **Status (C) and Quests (J/Q) screens.** Status shows your stats, reputation title, records and skill bars. Quests tracks progress and awards XP.
- **People.**
  - Mom and Dad are 3D characters who come out onto the porch.
  - Six townsfolk (Brittany, Gary, Linda, Cody, Jess, Ron) follow daily schedules across the four locations, shifted a little each day.
- **Environment.** Instanced Douglas-fir forest around every area, power poles with sagging wires, hydrants, street signs and lot lights.

## Round 4: graphics, UI and traffic
- **Lighting.**
  - Image-based lighting comes from PMREM environment maps baked from day, dusk and night gradient skies. It swaps with the time of day.
  - `envMapIntensity` is tuned per material: metals are full strength, glossy surfaces partial, matte surfaces low.
- **Atmosphere.** Distance fog is coloured to each sky (the sky domes are excluded from it). A drifting cloud layer is tinted and faded by time of day, and a subtle screen vignette darkens the edges.
- **Detail.**
  - Procedural normal maps on grass, asphalt and brick.
  - Shadows use a 4096 map on desktop.
  - Instanced grass tufts sway in the wind (9k per area on desktop, 3.5k on phones), avoiding roads, lots and buildings.
  - Leafy trees are built from jittered, lumpy clusters.
- **Building dressing.**
  - Fresh Basket: a parapet and rooftop AC units.
  - Ridgeway: awnings, gooseneck sign lamps that light at night, and planters.
  - ValuMart: an entrance canopy, bollards and cart corrals.
  - The house: gutters, downspouts and flower boxes.
- **Traffic (`traffic.js`).**
  - Every road has two lanes with three cars each, and the cars wrap around at the ends of the road.
  - Cars brake and honk if you're ahead of them in the lane, keep their distance from the car in front, and turn on their headlights at night.
  - If one hits you, you lose health based on its speed, get knocked back, and may black out.
  - There are now sidewalks on the far side of each road, plus crosswalks.
- **UI (`ui.js`).**
  - Frosted-glass HUD: a large clock with the place name, and colour-coded need bars with icons and values.
  - Top-right dock with icon buttons (Bag, Phone, Status, Quests) showing keyboard hints.
  - Interaction prompt is a pill with a highlighted key. Toasts and panels animate in.
  - Panels have a sticky header with a close button, a primary-button style, and styled shop rows, tabs, status grid and quest cards.
  - Esc opens a pause menu with Status, Quests, Bag, Controls, New game and the credit line.

## Round 5: weight, gym, dirt, laundry, park and storage

### Inventory and weight
- The bag is a 24-cell grid. Items can be dragged between cells: dropping on an empty cell moves the item, dropping on a different item swaps them, and dropping on the same item merges the stacks. Dragging to the strip below drops the item on the ground as a pickup.
- Every item has a weight. Capacity is 6 kg plus the backpack's bonus, plus 0.6 kg per Survival level and 0.2 kg per character level.
- Picking up something that pushes you past capacity fails with "Too heavy". If you end up over capacity anyway (for example after a pack comes off), you walk at half speed.
- Backpacks go in the Back slot and show on the character:
  - ValuMart: plastic bag (+2 kg, $0.10), daypack (+8 kg, $19.99), hiking pack (+18 kg, $54.99).
  - Ridgeway Outfitters: canvas rucksack (+12 kg, $34.99).
- The HUD shows carried kg instead of a can count.

### Iron Valley Fitness (gym)
- Open 5:00 to 23:00. Brandon at the front desk sells a $5 day pass (good until midnight) and a $20 membership that lasts 30 days.
- You can use up to 4 machines a day:
  - Treadmill trains Stamina.
  - Bench press trains Looks.
  - Weights train Looks and Stamina.
  - Rowing machine trains Stamina and Looks.
- Each workout skips about 40 to 45 minutes, uses energy, water and food, and adds 15 dirt.
- Stamina slows energy drain and raises walking speed. Looks reduces reputation losses and slowly regenerates reputation.
- A shower sets dirt to 0.

### Dirtiness and clothes
- Dirt rises about 35 points per day without a shower. Gross actions add more: picking up cans, digging in trash, touching or killing rats, eating scraps or rat meat, sleeping rough, fighting and breaking into cars.
- A shirt or pair of pants becomes a "dirty" version after 2 game days of wear. Shoes, hats and accessories never get dirty.
- Visible dirtiness is your dirt plus 18 for each dirty top, bottom or outer layer you're wearing.
- At visible dirtiness 60 or more:
  - Clerks refuse to serve you and automatic doors stay shut, except at the gym.
  - Reputation hits are larger and reputation slowly decays.
- A park restroom sink lowers dirt by 15.

### Suds & Duds laundromat
- Pam runs it, open 6:00 to 22:00.
- Load your dirty clothes into a washer for $2.50 and wait an hour of game time. Then move them to a dryer for $2 and wait another hour. Collect clean clothes at the end.
- The machines keep running if you walk away or ride the bus.

### Riverside Park
- Daily forage nodes: blackberries, apples and chanterelles. Chanterelles carry a 20% chance of making you sick.
- The park has a river with a footbridge, a restroom sink and quiet tree cover for camping.

### SecureStore indoor storage
- Walt at the desk rents you a unit for $20 per 30 days. Open 7:00 to 20:00.
- The units sit in hallways with roll-up doors. Open your unit to move items between it and your bag; a unit holds up to 150 kg.
- Renewal happens only in person at the desk.
- If the rent lapses, the contents go to auction and everything in the unit is lost.

### Bills and notifications
- The gym membership and storage rental each send a phone notification 3 days and 1 day before they run out.
- The gym also sends a notice when the membership expires, and storage sends one when the unit is auctioned.

## Round 6: signs, town edges, bus map, sky and time

### Store signs
- Power poles used to stand at the center of each storefront and cover the sign. They now run along the far side of the road and skip the middle of the block.
- The ValuMart sign sat behind the blue canopy band. It is now larger, sits on its own raised panel and is drawn in front of the band.

### Town edges (edges.js)
- Each location is described in road-relative coordinates: `u` runs along the road, `v` runs away from it. The table records the lot edges, the back line, the near sidewalk, the asphalt strip and where the buildings across the street start. Fresh Basket's road runs north–south, so its axes are swapped.
- **Boundaries.** A fence wraps the back and both sides of every lot. The fence fits the place:
  - wood privacy fence for houses, the gym and the laundromat
  - brick alley wall downtown
  - concrete wall behind ValuMart
  - chain-link along the highway
  - chain-link with barbed wire at the storage yard
  - wrought iron around the park
- **Across the street.** A continuous row of buildings runs about 150 m in both directions and blocks walking. The type depends on the place:
  - houses on residential streets
  - two-storey brick storefronts downtown
  - strip malls and warehouses on the highway
  - apartments by the gym and laundromat
  - warehouses in the industrial area
- **Neighbors.** Buildings of the same theme line the road past each side fence. Gaps between buildings are fenced.
- **Behind the lot.** Most places get a row of houses, brick backs, apartments or warehouses behind the back fence. The park keeps its woods.
- **Road ends.** About 30 m past each lot, "SIDEWALK CLOSED" sawhorses stand on both sidewalks, and an invisible blocker stops walking down the road. Cars still drive the full 280 m.
- **Storefront signs.** Most commercial buildings carry a made-up local business name. Some read FOR LEASE.
- **Ground.** A 300 × 340 m grass field sits under each place, with the road and sidewalks running to the fog line. This replaces the old ground disc, whose edge was visible.
- **Trees.** Firs that would sit inside buildings or the street corridor are removed by zero-scaling their instances. Trees stay as a backdrop behind the buildings.
- **Draw calls.** All edge geometry is merged per location and material into a few big meshes.
- **Draw distance.** The camera now draws out to 210 m, and fog runs from 35 to 165 m.

### Bus map and fares (busmap.js)
- A town map places the 8 stops at real 2D positions in miles, with:
  - roads: Hwy 26, Main St, Pine St, River Rd, Valley Rd, Industrial Way, 1st Ave and Oak St
  - the Deschutes River
  - the 14 Crosstown's route
- **Fare.** 25¢ for every started half mile of straight-line distance, with a 25¢ minimum. Examples:
  - Fresh Basket to Ridgeway Outfitters: 1.2 mi, 75¢
  - Pine Street to ValuMart: 5.1 mi, $2.75
- **Ride time.** Riding skips 4 minutes plus 3 minutes per mile.
- **Transfers.** Rides no longer hand out transfers. Any transfer you already have still covers one ride of any length.
- **Where to see it.** M or the Map button opens the map anywhere. Selecting a stop shows its distance, fare and ride time. At a bus stop, the same map is the ride menu, and tapping a stop or its row boards the bus.

### Night sky (skyfix.js)
- Stars were painted into the sky texture. They bunched up and streaked at the top of the dome, so they are removed from the texture.
- Stars are now 2,400 points spread evenly over the upper sky, with slight colour variation. They fade in with the night sky blend and drift slowly.
- The cloud layer used to be cut off along a hard line where its partial dome ended, with another seam where its texture wrapped. It now draws clouds across the wrap edge and fades them out toward the bottom of the dome.

### Time (timefix.js)
- Game time runs 6× real time while you play, so one game hour takes 10 real minutes. While the game is closed, time passes at the normal rate.
- Energy and dirt now change per game hour: energy runs out after 16 game hours, and dirt rises about 35 per game day. Food and thirst still empty in about 2 and 1.5 real hours (12 and 9 game hours).
- **Waiting.** The Wait option in the pause menu, or the T key, skips ahead:
  - 30 minutes, 1, 2, 4 or 8 hours
  - until 7 AM, or until 6 PM
- Waiting uses food, water and energy at a slightly reduced rate, adds dirt, and runs bill checks.
- A new game now starts at 9:30 AM.

### Bug fixed
- The skin shader named its noise values `m1`, `m2` and `m3`. three.js defines those names as macros in its environment-map code, so the shader failed to compile whenever reflections were on. They are renamed to `nz1`, `nz2` and `nz3`.

### Update notice (version.js)
- Each build gets a timestamp build ID, which is baked into `index.html` and written to `version.json` next to it.
- The game fetches `version.json` (bypassing the cache):
  - 15 seconds after loading
  - every 2 minutes
  - whenever the tab becomes visible again
- If the build ID differs from the running one, a banner slides up with Reload and Later buttons.
  - Reload saves the game first, then reloads the page.
  - Later hides the banner until a newer build appears.
- If `version.json` is missing, for example when the file is opened directly from disk, the check does nothing.

### Sidewalks, traffic direction and the house (round 7)
- **Sidewalks.** The near sidewalk on every street is 1.1 m wider. A raised concrete strip with a curb lip covers the old curb-side edge of the road: z 5.2–6.3 on the town streets, 16.2–17.3 at ValuMart and x 18–19.1 at Fresh Basket. You can now walk in front of every bus shelter without stepping into a lane.
- **Lanes.**
  - The two lanes moved over to fit the narrower road: 7.55 and 9.95 on the town streets, 18.55 and 20.95 at ValuMart, 20.55 and 23.5 at Fresh Basket.
  - Cars drive on the right, US-style. On the east–west streets the near lane now heads west and the far lane east.
  - A car only treats you as "in its lane" (honking and braking) once you step off the curb.
- **Bus.** The bus stops in the new near lane.
- **Cars facing backwards.** The car body shapes were built with the hood pointing at −Z, while the headlights, mirrors and direction of travel use +Z. So every car, parked or moving, looked like it was driving backwards. The bodies are now rotated by −90° instead of +90°, which puts the hood at the front.
- **House.** The vending machine outside the parents' house is gone.
