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

### Controller support (gamepad.js)
- Uses the browser Gamepad API with the standard button layout (Xbox naming below; PlayStation buttons map by position). The game polls the controller every frame.
- **Taking over from mouse and keyboard.** The first stick or button input switches to controller mode:
  - pointer lock isn't needed
  - the interact prompt shows **A** instead of **E**
- **Switching back.** Moving the mouse or pressing a real key while no menu is open switches back.
- **Walking around:**
  - **Left stick:** walk. It feeds the same movement vector as the touch joystick. Click the stick to toggle running (it turns off when you let go of the stick), or hold LT to run.
  - **Right stick:** look, with a 1.6 power curve and an 18% dead zone.
  - **A:** interact.
  - **X or RT:** hit.
  - **Y:** bag.
  - **LB:** map.
  - **RB:** phone.
  - **View:** quests.
  - **R3 or D-pad down:** status.
  - **D-pad up:** wait.
  - **Menu:** pause or resume.
- **In menus** (title screen, pause screen, panels, bag, map):
  - **D-pad or left stick:** move a highlighted focus ring to the nearest button, slot or map stop in that direction. Holding repeats after a short delay.
  - **A:** activate the highlighted item.
  - **B:** back or close.
  - **Right stick:** scroll.
  - **In the bag:** A selects a slot. X picks the item up (it's highlighted in blue), and X or A on another slot swaps the two.
- **Rumble.** Getting hit by a car makes the controller rumble when it supports that.
- **P key.** P now opens the phone from the keyboard too, matching the hint on the dock button.
- **Map tweak.** The ValuMart stop moved slightly west and its label sits under the dot, so it no longer overlaps the gym's fare.

## Round 8: loading, storage, controller polish and fixes

### Loading screen (loading.js)
- A full-screen loading card covers the scene in two cases:
  - **Pressing play.** It reads "Loading" with the place name, and the subtitle says "Starting a new life" or "Picking up where you left off".
  - **Riding the bus.** It reads "Riding the 14 Crosstown" with the destination.
- Each card shows a progress bar and a rotating gameplay tip.
- **When it hides.** The card stays up until all of these are true, or for 22 s at most:
  - the player model is built
  - the delayed clerk spawns are done (about 11.5 s after the page loads)
  - a few frames have rendered
  - 12 frames in a row came in under 75 ms
  - a minimum time has passed (1.8 s on start, 1.4 s on the bus)
- **Movement.** The player is held in place while the card is up.
- **No shader pre-compile.** An earlier version called `renderer.compile` on the whole scene. That compiles every location at once and stalled software-rendered browsers for seconds, so it was dropped. The loader now just waits for frames to settle while the scene renders behind it.
- **Bug fixed: blank world.** The loader's frame callback was first named `frame`. Function declarations inside a block are hoisted to the enclosing scope, so it replaced the game's main `frame()` loop and the world went black. It is now `ldFrame`. The gamepad module's `back` and `loop` were renamed for the same reason.
- **Bug fixed: bus stop picking a destination by itself.**
  - The cause: clicking the bus stop released pointer lock and opened the map, and the same click's mouse-up then landed on whichever fare row sat under the cursor.
  - The fix: for 400 ms after any panel opens, clicks and mouse-ups inside it are ignored.

### New game without reloading (newgame.js)
- **Start over** in the pause menu now asks for confirmation in a game panel instead of a browser dialog, so it works with a controller. The **Start over** button on the game-over screen uses the same reset.
- **What the reset clears:**
  - every `currentyear-*` save key
  - needs, bag and equipment, jobs, email, stats, quests, skills and XP
  - gym membership and day pass
  - storage unit and laundry machines
  - foraging, trash and car timers
  - dropped items and the tent
- **What it sets up fresh:** you're back on Pine Street at 9:30 AM, with new cans to collect.
- **Then:** the game returns to the title screen. The opening scene and loading card run again when you press play.

### Startup failures
- **Fallback downloads.** If three.js doesn't load from cdnjs, the page tries jsDelivr, then unpkg.
- **Visible errors.** If three.js still fails, or the browser can't create a WebGL context, a visible error card explains what happened and offers **Try again**.
  - Before, the message went into title text that is hidden. The player saw a bare play button that did nothing, which matches the mobile screenshot after a game over.
- **Graphics reset.** If the phone drops the WebGL context to reclaim memory, the game saves and shows a "Graphics paused" card with a Continue button.
- **Narrow screens.** Under 640 px, the top-right buttons wrap into two rows and drop their keyboard hints, so they no longer cover the clock and needs panel.

### SecureStore rebuild (places.js, stor2.js)
- **Size.** The building is now 40 × 48 m and 5.2 m tall. It has a lobby across the front and a partition wall with a doorway leading back to the hallways.
- **Hallways.** Four hallways run front to back, joined by a cross hallway at the back. There are 80 roll-up units, numbered 101–110 through 801–810, each with door rails and a number plate.
- **Lobby:**
  - an L-shaped counter with a laminate top, a logo panel, a monitor, a bell and paperwork
  - a back shelf of moving boxes and a pricing poster
  - waiting chairs, plants and a hand truck
  - several ceiling lights that dim when the office closes
- **Outside.** A striped side parking lot. The street's edge scenery was widened to fit.
- **Walt sells nothing now.** Talking to him, or using the front desk, opens a conversation:
  - **Read the rental agreement** shows a paper-style agreement with a randomly picked unit number, the rent, the paid-through date, the rules and the lien-auction clause, plus signature lines.
  - **Sign and pay $20.00** rents the unit and gives you a "SecureStore rental agreement" item (10 g).
  - Once you're a tenant, you can pay another month, close out the unit (which takes back the agreement), ask for a replacement copy, or ask about a job (his normal job-application conversation, with the Shop button removed).
  - If you're too dirty to be served, he gives the normal refusal.

### Weight display
- Every item already had a weight, but anything under 0.05 kg showed as "0.0". Weights under 1 kg now show in grams, in the bag cells, the item details and the carry bar.
- Paper items now have their own weights: transfer 2 g, citation 5 g, applications 12 g, pen 11 g.

### Caps
- The baked cap brim was far too wide, so it wrapped around the head like a bucket hat and sat at eye level.
- The brim is now about 40% of the hat rim's width, sticks out about 8.5 cm, curves down at the sides and sits higher on the forehead.

### First-person punch (hand.js)
- **The model.** A built fist (palm, knuckles, folded fingers and thumb), a wrist and forearm, and a sleeve with a cuff. The sleeve is tinted to the outer layer you're wearing, or your top.
- **Unarmed.** The arm jabs in from the lower right toward the center of the screen, then pulls back, and is only shown during a swing.
- **With a weapon.** The same fist and forearm replace the old ball "fist" and grip the weapon.

### Phone menu (phoneui.js) and starting money
- **One button.** The top-right dock now has only the **Phone** button, which carries the unread-mail badge. Bag, Map, Status and Quests no longer have their own buttons. Your money and carried weight moved to a line at the bottom of the needs panel.
- **Home screen.** The Phone button (P, or RB on a controller) opens a smartphone with a status bar, a clock-and-place widget and eight apps:
  - **Mail:** the inbox.
  - **Map:** the bus map.
  - **Bag:** inventory and clothing.
  - **Status.**
  - **Quests.**
  - **Clock:** wait.
  - **Wallet:** cash, cans and what they're worth, gym and storage status, applications.
  - **Settings:** controls, pause, start over and credits.
- **Apps.** An app opens inside the same phone frame, with a **‹ Home** button and a home bar to return to the home screen. The ✕ button closes everything.
- **Other menus.** Conversations and shops opened in the world keep their normal panel style.
- **Keyboard shortcuts.** I, M, C, J and T still open their menus directly.
- **Controller.** In the controller's menu navigation, the phone screen counts as a menu: RB toggles the phone and B closes it.
- **Starting money.** Every new game, including after Start over, begins with $20. Mom still mentions the money in her speech, but she no longer hands it over at the end, so skipping the speech doesn't cost you $20.

### Phone apps look like phone apps (phoneui2.js)
- **The look.** Apps opened from the phone use a dark smartphone style:
  - a fake status bar with the game time and "4G"
  - a "‹ Home" back link in the app's accent colour
  - a large bold title, with the cash shown in green
  - system fonts
  - grouped rounded lists for buttons, stats, mail, quests and fares, with › arrows
  - a segmented control for the bag tabs
  - the footer hidden
- **Accent colours.** Each app has its own: Mail blue, Map green, Bag orange, Status purple, Quests yellow, Clock orange, Wallet green, Settings blue.
- **Closing an app goes back to the phone home screen.** This covers ✕, Close, Bye, Escape and tapping outside. Opening another panel from inside an app keeps the phone style. You leave the phone from its home screen: the home bar, tapping outside, P or Escape.
- **Settings** is laid out like a phone's Settings app:
  - a profile card: level, day, place
  - **Sound & display:** Sound on/off, Show frame rate on/off, and a Look sensitivity slider from 40% to 200% (applies to touch, mouse and controller)
  - **Game:** Controls, Pause game, Start over (in red)
  - **About:** the credit line, Website and Patreon
- **Saved settings.** Settings are saved under `cy-settings`, outside the `currentyear-*` save keys, so Start over keeps them.

## Round 9: survival meters, collapse and death (survival.js)
- **HUD.** The needs panel is gone from the main screen. Only the clock and the Phone button remain.
- **Status app.** The meters now live at the top of Status as bars: Health, Food, Water, Energy, Dirtiness and Reputation. They update live, and a meter turns red when it's low (or, for Dirtiness, when it's high).
- **No money in phone apps.** Phone apps no longer show your money in their header. Use the Wallet app.
- **Warnings.** Notification-style banners slide down from the top, with a sound and a short phone vibration where supported:
  - **Food and water:** at 30%, 15%, 5% and 0%.
  - **Energy:** at 30%, 15% and 5%.
  - **Health:** at 50%, 25% and 10%.
  - **Dirtiness:** at 60%.
- Only the most serious new warning shows, and a level re-arms once the meter recovers 6 points above it. Critical warnings are red and stay up longer. Tapping a banner opens Status in the phone. The old plain-text need warnings are silenced.
- **Starving or dehydrated.** At 0 food you lose 25 health per game hour. At 0 water you lose 35. Health only regenerates while food and water are both above 20.
- **Death.** If health reaches 0 while food or water is at 0, you die:
  - every save key is erased immediately and further saves are blocked
  - a "You died" screen names the cause and how many days you lasted
  - **Start over** runs the in-place new game (no reload)
  - Health lost in a fight still knocks you out instead, as before.
- **Collapsing.** Energy at 0 makes you collapse:
  - the camera tips to the ground and you sleep 9–12 hours
  - you lose food and water as you would sleeping, and gain a little dirt
- **Collapsing indoors** (any store; the parents' porch counts as outside): staff find you within the hour. Sheriff citation of $50, −6 reputation, and you're moved outside the door.
- **Collapsing outside.** Each hour has a chance of someone finding you, based on foot traffic nearby:
  - if found, you're woken early with only partial energy, plus a $35 citation and −5 reputation
  - if nobody finds you, you wake with full energy
- **Punching hand.** The fist was rebuilt from smooth shapes: a rounded palm, capsule-shaped fingers bent at the knuckles, and a thumb wrapped across the front. The forearm and sleeve are smooth tapered shapes with a rolled cuff, and the skin tone is warmer.

### Fix: back-row buildings poking into lots
- **The bug.** The row of buildings behind each lot was built facing the wrong way. Each building started at the back line and extended forward into the lot by its depth (up to 22 m), instead of extending away from it. At Fresh Basket, a warehouse stood beside the storefront and hid half the sign. Other locations had the same overlap at the back of their lots.
- **The fix.** These buildings now face the lot and extend away from it, behind the back fence.

## Round 10: walking around cars, fight reactions, misc fixes
- **NPCs and parked cars** (npcwalk.js):
  - Parked car bounding boxes are merged into clusters (rows of cars become one block).
  - When the straight line to a walker's next waypoint crosses a cluster, the walker detours via the nearest clear corner, then continues.
  - A waypoint that sits inside a car cluster is skipped.
  - Walkers also stop and wait when a moving car is in, or about to enter, the spot just ahead of them.
  - This applies to shoppers, townies and night guards. In a test of 40 random crossings, 39 got through without touching a car; the other was still walking when the test stopped, but never entered a car.
- **Hitting people:**
  - Punches only test against people and rats now, so you can reach a clerk across a counter.
  - On the first hit the person turns to face you, and you see "X fights back!" or "X runs away!".
  - Fighters raise their guard and throw a visible punch.
  - While someone is fighting, fleeing or knocked out, you can't talk to them. Talking comes back once they reset.
- **Conversations no longer show your money.**
- **Phone:** a large "Close phone" button sits above the home bar on the home screen.
- **Eyebrows:** the hair-strand eyebrow layers were removed; they stuck out at odd angles on some faces. Eyebrows are now only drawn into the skin, where they follow each face's eye positions.
- **Flickering surfaces:**
  - The ValuMart roof's front edge and the blue band in front of it shared the same plane. The band now sits further forward, and the sign sits in front of the band.
  - The camera's near clipping distance went from 0.05 to 0.12, which roughly doubles depth precision on phones.
  - Fake storefronts on the street-side buildings sit further off their walls.

## Round 11: laundromat, crowds, ValuMart, bus
- **Suds & Duds:**
  - **Building:** rebuilt at 20 × 18 m. The front wall now closes fully around the door, with no gaps beside the header. The storefront has window mullions.
  - **Hours:** open 24 hours, with an "OPEN 24 HRS" neon sign.
  - **Machines:** 23 washers along the left wall and on a central back-to-back island, and 24 dryers (two-high stacks along the back wall plus a row on the right wall).
  - **Furniture and fittings:** folding tables, two rows of plastic chairs by the windows, laundry carts, a wall TV, a soap vending machine, a change machine and a house-rules sign.
  - **No clerk:** Pam is gone, so there's nothing to buy over a counter; machines are paid at the machine. The snack vending machine stays out front.
  - **Parking:** the lot moved to the right side.
- **Crowds:** 14 more townies, each with a daily schedule, so every location has people at most hours:
  - Tammy, Earl, Destiny, Shelby and Darlene keep the laundromat busy around the clock.
  - Wade, Shelby, Travis, Darlene, Destiny, Kelsey, Dusty and Amber shop at ValuMart.
  - Randy walks the storage hallways.
  - Hank walks Pine Street.
  - Marge and Dean are at the park.
  - Others split their day between the gym, Ridgeway, Fresh Basket and its lot.
  - SecureStore got a walking path through its lobby and hallways.
- **ValuMart's empty right side:**
  - a shoe wall
  - four round clothing racks
  - an electronics wall with ten glowing TVs
  - pallets of water and pet food
  - a "Fall Harvest" pumpkin table
  - a fishing section with rods and tackle boxes (for the coming pier)
  - a row of bikes
- **Bus:**
  - **Body:** rebuilt with rounded extruded panels, a tinted window band with pillars, a cream roof stripe and an AC pod, a slanted windshield, bumpers, mirrors on arms, dark wheel arches, and wheels with chrome hubs.
  - **Doors:** two real double doors on the curb side, each with framed glass leaves, rubber edges, a grab bar and a step. The doors swing open whenever the bus has stopped and close as it pulls away.

## Round 12: crouching, being seen, hiding from the law (stealth.js)
- **Crouch:**
  - Toggle with Z, or hold Ctrl. On a controller, B in-game. On touch screens, a round Crouch button above Hit.
  - Your eye height drops from 1.62 m to 1.02 m and you move at 42% speed.
- **Who can see you.** A person sees you if all of these hold:
  - They're within range: 24 m standing (18 m indoors), or 9 m crouched (8 m indoors).
  - You're inside their view cone: wide when you're standing, narrow when you're crouched. Anyone within 1.6 m notices you regardless.
  - Nothing solid is between you. Shelves, counters, cars, buildings and fences all count.
- **What crouching affects.** The witness check used by gross actions, theft, attacks and break-ins uses the same rule while you're crouched. You can dig in trash or swing at someone unseen if you sneak.
- **Eye badge.** While you're crouched or a deputy is searching, a badge under the crosshair shows a closed eye and "Hidden", or an open eye and who can see you (amber for a civilian, red for the deputy). During a search it also shows the seconds left and a fill bar as the deputy locks on.
- **Cops search instead of arresting instantly.** When the call-the-cops countdown ends:
  - Deputy Hall (a visible deputy in a tan uniform) walks in from the bus stop, and a 28-second search starts.
  - He walks to where you were, then picks random spots within 13 m of it, pauses and looks around.
  - If he keeps you in sight for 0.6 s (1.2 s if you're crouched), or gets within 1.6 m, you're arrested as before.
  - If you stay hidden until the timer runs out, or leave the area by bus, he gives up: −2 reputation and a little XP.

## Round 13: sound, wildlife, walking between places
- **Audio** (audio.js), all synthesized live with WebAudio. It starts on the first tap or key press.
  - **Music:** slow pads built from four-chord cycles, with sparse plucked notes and a soft tape-delay echo. The mood follows the place and time:
    - day: bright major sevenths
    - evening: warm minor
    - night: dark minor ninths
    - indoors: gentle store tones through a muffled filter
  - **Ambience:** bird chirps by day and crickets at night outdoors, a low wind rumble outside, and a fluorescent hum indoors.
  - **Footsteps:** faint steps every stride. They sound different on tile, asphalt and grass, and quieter when crouched.
  - **Talking:** any "Name: “…”" line, and every clerk conversation, plays a short babble of formant blips. Women get a higher pitch, and longer lines get more syllables.
  - **More effects:** crunching when eating, gulping when drinking, automatic doors whooshing open, and birds flapping off.
  - **Settings:** the Settings app has Music and Ambient sounds switches, on top of the master Sound switch.
- **Wildlife** (wildlife.js):
  - by day, a flock of nine birds crosses the sky every half minute or so
  - five birds peck on the ground near each lot (or on the park lawn) and scatter when you come within 4.5 m, returning later
  - after dark, six bats flit overhead
  - flies buzz over nearby trash cans and dumpsters from 8 AM to 8 PM
  - fireflies glow at night in the park and on Pine Street
- **Walking between places** (walkedge.js):
  - Reaching the "SIDEWALK CLOSED" barricade at either end of a street asks whether to walk to the neighbouring location.
  - The neighbours follow the street order: Fresh Basket, Pine Street, Ridgeway, ValuMart, the gym, the laundromat, the park, then storage.
  - The prompt shows the distance, the walking time (20 min per mile), the energy cost (7 per mile) and what the bus would cost.
  - Walking skips that time, costs energy, food, water and a little dirt, trains Stamina, and drops you at the destination's bus stop.
- **Bug fixed:** a function named `update` inside the wildlife block replaced the game's own `update` loop through block-scope hoisting, and the camera froze. It's now `wlUpdate`, and every module was checked for the same kind of clash.

## Round 14: park playground, hobo camp, begging
- **Park benches** now sit beside the paths instead of on them.
- **Playground** (park2.js), west of the park's center path:
  - wood-chip ground with a low steel fence and a PLAYGROUND sign
  - a two-seat swing set that really swings, a slide with a ladder, a sandbox and a spring rider
  - grass no longer grows through the chips
- **Kids** play there from 9 AM to 7 PM: Tyler Jr. on the swing, Lily and Mason running laps around the equipment and pausing now and then.
  - They're 62% scale with slightly larger heads.
  - You can say hi, but they can't be targeted by punches (the attack raycast only hits combatants and rats).
- **Hobo camp** on the park's east side by the river: two tents, a blue tarp on poles, a burn barrel with a flickering fire (brighter at night), shopping carts full of junk, crates and trash bags.
  - **Old Pete** buys rat meat at $1.50 each and teaches you to make a begging sign.
  - **Skeeter** sells mystery pills ($6) and jars of hooch ($3), and buys rat meat for $1.25. Buying in front of witnesses costs 3 reputation.
  - Both talk without showing your money.
- **Pills** give +45 energy right away; about 90 game minutes later the comedown hits for −38 energy and −8 health.
- **Hooch** is a drink: a little water and +18 energy.
- **Begging:**
  - Flattened cardboard now turns up in trash cans.
  - Once Pete has taught you, cardboard plus a pen makes a "Cardboard sign" (inventory button).
  - Using the sign sits you down outdoors with it held up. Every 5–9 seconds there's a chance someone gives 25¢–$5.
  - The chance grows with how dirty you look and with how many people and cars are nearby. Under 40 dirt it's nearly zero and people tell you to get a job.
  - It costs a point of reputation every few rounds. Moving or tapping the pill ends it. You can't beg indoors.
- **Inventory:** any item can now define its own action button (used by cardboard, the sign and pills).
