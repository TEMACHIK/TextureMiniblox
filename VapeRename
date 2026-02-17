/**
 * @type {Record<string | RegExp, string>}
 */
let replacements = {};
let dumpedVarNames = {};
const storeName = "a" + crypto.randomUUID().replaceAll("-", "").substring(16);
const vapeName = crypto.randomUUID().replaceAll("-", "").substring(16);
const VERSION = "3.1.2.1";

// ANTICHEAT HOOK
function replaceAndCopyFunction(oldFunc, newFunc) {
	return new Proxy(oldFunc, {
		apply(orig, origIden, origArgs) {
			const result = orig.apply(origIden, origArgs);
			newFunc(result);
			return result;
		},
		get(orig) {
			return orig;
		}
	});
}

Object.getOwnPropertyNames = replaceAndCopyFunction(Object.getOwnPropertyNames, function (list) {
	if (list.indexOf(storeName) != -1) list.splice(list.indexOf(storeName), 1);
	return list;
});
Object.getOwnPropertyDescriptors = replaceAndCopyFunction(Object.getOwnPropertyDescriptors, function (list) {
	delete list[storeName];
	return list;
});

/**
 *
 * @param {string} replacement
 * @param {string} code
 * @param {boolean} replace
 */
function addModification(replacement, code, replace) {
	replacements[replacement] = [code, replace];
}

function addDump(replacement, code) {
	dumpedVarNames[replacement] = code;
}

/**
 *
 * @param {string} text
 */
function modifyCode(text) {
	let modifiedText = text;
	for (const [name, regex] of Object.entries(dumpedVarNames)) {
		const matched = modifiedText.match(regex);
		if (matched) {
			for (const [replacement, code] of Object.entries(replacements)) {
				delete replacements[replacement];
				replacements[replacement.replaceAll(name, matched[1])] = [code[0].replaceAll(name, matched[1]), code[1]];
			}
		}
	}
	const unmatchedDumps = Object.entries(dumpedVarNames).filter(e => !modifiedText.match(e[1]));
	if (unmatchedDumps.length > 0) console.warn("Unmatched dumps:", unmatchedDumps);

	const unmatchedReplacements = Object.entries(replacements).filter(r => modifiedText.replace(r[0]) === text);
	if (unmatchedReplacements.length > 0) console.warn("Unmatched replacements:", unmatchedReplacements);

	for (const [replacement, code] of Object.entries(replacements)) {
		modifiedText = modifiedText.replace(replacement, code[1] ? code[0] : replacement + code[0]);
		// TODO: handle the 2nd occurrence, which inside a string in a varible called "jsContent".
		// (screw you vector)
	}

	const newScript = document.createElement("script");
	newScript.type = "module";
	newScript.crossOrigin = "";
	newScript.textContent = modifiedText;
	const head = document.querySelector("head");
	head.appendChild(newScript);
	newScript.textContent = "";
	newScript.remove();
}

(function () {
	'use strict';

	// DUMPING
	addDump('moveStrafeDump', 'this\\.([a-zA-Z]+)=\\([a-zA-Z]+\\.right');
	addDump('moveForwardDump', 'this\\.([a-zA-Z]+)=\\([a-zA-Z]+\\.(up|down)');
	addDump('keyPressedDump', 'function ([a-zA-Z]*)\\([a-zA-Z]*\\)\{return keyPressed\\([a-zA-Z]*\\)');
	addDump('entitiesDump', 'this\.([a-zA-Z]*)\.values\\(\\)\\)[a-zA-Z]* instanceof EntityTNTPrimed');
	addDump('isInvisibleDump', '[a-zA-Z]*\.([a-zA-Z]*)\\(\\)\\)&&\\([a-zA-Z]*=new ([a-zA-Z]*)\\(new');
	addDump('attackDump', 'hitVec.z\}\\)\}\\)\\),player\.([a-zA-Z]*)');
	addDump('lastReportedYawDump', 'this\.([a-zA-Z]*)=this\.yaw,this\.last');
	addDump('windowClickDump', '([a-zA-Z]*)\\(this\.inventorySlots\.windowId');
	addDump('playerControllerDump', 'const ([a-zA-Z]*)=new PlayerController,');
	addDump('damageReduceAmountDump', 'ItemArmor&&\\([a-zA-Z]*\\+\\=[a-zA-Z]*\.([a-zA-Z]*)');
	addDump('boxGeometryDump', 'w=new Mesh\\(new ([a-zA-Z]*)\\(1');
	addDump('syncItemDump', 'playerControllerMP\.([a-zA-Z]*)\\(\\),ClientSocket\.sendPacket');

	// PRE
	addModification("u.serverInfo.permissionLevel>=PermissionLevel.ADMIN", "||adminSpoof.enabled");
	// I was trying to make stop server work but uh it didn't work, I think they actually check.
	//addModification("u.serverInfo.permissionLevel===PermissionLevel.OWNER", "||adminSpoof.enabled");
	addModification(
		"this.serverInfo.handlePacket(x.serverInfo),",
		`!enabledModules['AutoPrivate'] ||
(ClientSocket.sendPacket(new SPacketAdminAction({action: {
		case: "updateAccessControl",
		value: {
			accessControl: "private"
		}
}})), toast({
	title: "Set access control to private!",
	status: "success"
})),`
	);

	addModification('document.addEventListener("DOMContentLoaded",startGame,!1);', `
		setTimeout(function() {
			var DOMContentLoaded_event = document.createEvent("Event");
			DOMContentLoaded_event.initEvent("DOMContentLoaded", true, true);
			document.dispatchEvent(DOMContentLoaded_event);
		}, 0);
	`);
	addModification("y:this.getEntityBoundingBox().min.y,", `y: sendY != false
? sendY
: enabledModules["AutoClip"]
	? handleAutoClip(this.pos, this.getEyeHeight(), this.width, this.height)
: this.getEntityBoundingBox().min.y,`, true);
	addModification("const player=new ClientEntityPlayer", `
// note: when using desync,
// your position will only update every 20 ticks.
let serverPos = player.pos.clone();
`);
	addModification('Potions.jump.getId(),"5");', `
		let adminSpoof;
		let blocking = false;
		let sendYaw = false;
		let sendY = false;
		let desync = false;
		let sendGround;
		let breakStart = Date.now();
		let noMove = Date.now();

		// a list of miniblox usernames to not attack / ignore
		/** @type string[] **/
		const friends = [];
		let ignoreFriends = false;

		let enabledModules = {};
		let modules = {};

		let keybindCallbacks = {};
		let keybindList = {};

		let tickLoop = {};
		let renderTickLoop = {};

		let lastJoined, velocityhori, velocityvert, chatdisablermsg, textguifont, textguisize, textguishadow, attackedEntity, stepheight;
		let useAccountGen, accountGenEndpoint
		let attackTime = Date.now();
		let chatDelay = Date.now();

		/**
		 * clamps the given position to the given range
		 * @param {Vector3} pos
		 * @param {Vector3} serverPos
		 * @param {number} range
		 * @returns {Vector3} the clamped position
		**/
		function desyncMath(pos, serverPos, range) {
			const moveVec = {x: (pos.x - serverPos.x), y: (pos.y - serverPos.y), z: (pos.z - serverPos.z)};
			const moveMag = Math.sqrt(moveVec.x * moveVec.x + moveVec.y * moveVec.y + moveVec.z * moveVec.z);

			return moveMag > range ? {
				x: serverPos.x + ((moveVec.x / moveMag) * range),
				y: serverPos.y + ((moveVec.y / moveMag) * range),
				z: serverPos.z + ((moveVec.z / moveMag) * range)
			} : pos;
		}

		async function generateAccount() {
			toast({
				title: "generating miniblox account via integration...",
				status: "info",
				duration: 0.3e3
			});
			const res = await fetch(accountGenEndpoint[1]);
			if (!res.ok)
				throw await res.text();
			const j = await res.json();
			toast({
				title: \`Generated miniblox account! named \${j.name}!\`,
				status: "success",
				duration: 1e3
			});
			return j;
		}

		function getModule(s) {
			for(const [n, m] of Object.entries(modules)) {
				if (n.toLocaleLowerCase() == s.toLocaleLowerCase()) return m;
			}
		}
// UNCOMMENT FOR 1.7 BLOCKING EXPERIMENT (doesn't work)
/*
		function applySwingOffset(matrix, swingProgress) {
			const f = Math.sin(swingProgress * swingProgress * Math.PI);
			const g = Math.sin(Math.sqrt(swingProgress) * Math.PI);

			// Create temporary matrices for each rotation
			const rotY1 = new Matrix4().makeRotationY(-1 * (45.0 + f * -20.0) * Math.PI / 180);
			const rotZ = new Matrix4().makeRotationZ(-1 * g * -20.0 * Math.PI / 180);
			const rotX = new Matrix4().makeRotationX(g * -80.0 * Math.PI / 180);
			const rotY2 = new Matrix4().makeRotationY(-1 * -45.0 * Math.PI / 180);
			console.log("swing progress:", swingProgress);

			// Apply rotations in the same order as the original code
			matrix.multiply(rotY1);
			matrix.multiply(rotZ);
			matrix.multiply(rotX);
			matrix.multiply(rotY2);
		}

		function handleBlockingAnimation(h3d, swingProgress) {
			applySwingOffset(h3d.position, swingProgress);
		}
*/

		let j;
		for (j = 0; j < 26; j++) keybindList[j + 65] = keybindList["Key" + String.fromCharCode(j + 65)] = String.fromCharCode(j + 97);
		for (j = 0; j < 10; j++) keybindList[48 + j] = keybindList["Digit" + j] = "" + j;
		window.addEventListener("keydown", function(key) {
			const func = keybindCallbacks[keybindList[key.code]];
			if (func) func(key);
		});
	`);

	addModification('VERSION$1," | ",', `"${vapeName} v${VERSION}"," | ",`);
	addModification('if(!x.canConnect){', 'x.errorMessage = x.errorMessage === "Could not join server. You are connected to a VPN or proxy. Please disconnect from it and refresh the page." ? "[Vape] You\'re IP banned (these probably don\'t exist now anyways)" : x.errorMessage;');

	// DRAWING SETUP
	addModification('I(this,"glintTexture");', `
		I(this, "vapeTexture");
		I(this, "v4Texture");
	`);
	/**
	 * @param {string} url
	 * @returns
	 */
	const corsMoment = url => {
		return new URL(`https://corsproxy.io/?url=${url}`).href;
	}
	addModification('skinManager.loadTextures(),', ',this.loadVape(),');
	addModification('async loadSpritesheet(){', `
		async loadVape() {
			this.vapeTexture = await this.loader.loadAsync("${corsMoment("https://codeberg.org/RealPacket/VapeForMiniblox/raw/branch/main/assets/logo.png")}");
			this.v4Texture = await this.loader.loadAsync("${corsMoment("https://codeberg.org/RealPacket/VapeForMiniblox/raw/branch/main/assets/logov4.png")}");
		}
		async loadSpritesheet(){
	`, true);

	// TELEPORT FIX
	addModification('player.setPositionAndRotation(h.x,h.y,h.z,h.yaw,h.pitch),', `
		noMove = Date.now() + 500;
		player.setPositionAndRotation(h.x,h.y,h.z,h.yaw,h.pitch),
	`, true);


	// PREDICTION AC FIXER (make the ac a bit less annoying (e.g. when scaffolding))
	// ig but this should be done in the desync branch instead lol
	// 	addModification("if(h.reset){this.setPosition(h.x,h.y,h.z),this.reset();return}", "", true);
	// 	addModification("this.serverDistance=y", `
	// if (h.reset) {
	// 	if (this.serverDistance >= 4) {
	// 		this.setPosition(h.x, h.y, h.z);
	// 	} else {
	// 		ClientSocket.sendPacket(new SPacketPlayerInput({sequenceNumber: NaN, pos: new PBVector3(g)}));
	// 		ClientSocket.sendPacket(new SPacketPlayerInput({sequenceNumber: NaN, pos: new PBVector3({x: h.x + 8, ...h})}));
	// 	}
	// 	this.reset();
	// 	return;
	// }
	// `);

	addModification('COLOR_TOOLTIP_BG,BORDER_SIZE)}', `
		function drawImage(ctx, img, posX, posY, sizeX, sizeY, color) {
			if (color) {
				ctx.fillStyle = color;
				ctx.fillRect(posX, posY, sizeX, sizeY);
				ctx.globalCompositeOperation = "destination-in";
			}
			ctx.drawImage(img, posX, posY, sizeX, sizeY);
			if (color) ctx.globalCompositeOperation = "source-over";
		}
	`);

	// TEXT GUI
	addModification('(this.drawSelectedItemStack(),this.drawHintBox())', /*js*/`
		if (ctx$5 && enabledModules["TextGUI"]) {
			const canvasW = ctx$5.canvas.width;
			const canvasH = ctx$5.canvas.height;
			const colorOffset = (Date.now() / 4000);
			const posX = 15;
			const posY = 17;
			ctx$5.imageSmoothingEnabled = true;
			ctx$5.imageSmoothingQuality = "high";
			drawImage(ctx$5, textureManager.vapeTexture.image, posX, posY, 80, 21, \`HSL(\${(colorOffset % 1) * 360}, 100%, 50%)\`);
			drawImage(ctx$5, textureManager.v4Texture.image, posX + 81, posY + 1, 33, 18);

			let offset = 0;
			let filtered = Object.values(modules).filter(m => m.enabled && m.name !== "TextGUI");

			filtered.sort((a, b) => {
				const aName = a.name;
				const bName = b.name;
				const compA = ctx$5.measureText(aName).width;
				const compB = ctx$5.measureText(bName).width;
				return compA < compB ? 1 : -1;
			});

			for(const module of filtered) {
				offset++;
				
				const fontStyle = \`\${textguisize[1]}px \${textguifont[1]}\`;
				ctx$5.font = fontStyle;

				// Build strings
				const rainbowText = module.name;
				const modeText = module.tag?.trim();

				const fullText = \`\${rainbowText}\${modeText ? " " + modeText : ""}\`;
				const textWidth = ctx$5.measureText(fullText).width;
				const x = canvasW - textWidth - posX;
				const y = posY + (textguisize[1] + 3) * offset;

				// Shadow for both parts
				ctx$5.shadowColor = "black";
				ctx$5.shadowBlur = 4;
				ctx$5.shadowOffsetX = 1;
				ctx$5.shadowOffsetY = 1;

				// Draw rainbow part
				drawText(
					ctx$5,
					rainbowText,
					x,
					y,
					fontStyle,
					\`hsl(\${((colorOffset - 0.025 * offset) % 1) * 360},100%,50%)\`,
					"left",
					"top",
					1,
					textguishadow[1]
				);

				// Draw grey mode part (after rainbow width)
				if (modeText) {
					const rainbowWidth = ctx$5.measureText(rainbowText).width;
					drawText(
						ctx$5,
						modeText,
						x + rainbowWidth + 4,
						y,
						fontStyle,
						"#bbbbbb",
						"left",
						"top",
						1,
						textguishadow[1]
					);
				}

				// Reset shadow
				ctx$5.shadowColor = "transparent";
				ctx$5.shadowBlur = 0;
				ctx$5.shadowOffsetX = 0;
				ctx$5.shadowOffsetY = 0;
			}
		}
	`);

	// HOOKS
	// instructions because this replacement is very vague when trying to find it after an update:
	// 1. search for "moveFlying("
	// 2. select the first result
	// 3. look for "this.motion.z+="
	// 4. use that as the replacement
	// thanks GOD that I had the old bundle to find this
	addModification('+=h*y+u*x}', `
		if (this == player) {
			for(const [index, func] of Object.entries(tickLoop)) if (func) func();
		}
	`);
	addModification('this.game.unleash.isEnabled("disable-ads")', 'true', true);
	// in EntityManager, renderEntities function
	addModification('h.render()})', '; for(const [index, func] of Object.entries(renderTickLoop)) if (func) func();');
	addModification('updateNameTag(){let h="white",p=1;', 'this.entity.team = this.entity.profile.cosmetics.color;');
	addModification('connect(u,h=!1,p=!1){', 'lastJoined = u;');
	addModification('SliderOption("Render Distance ",2,8,3)', 'SliderOption("Render Distance ",2,64,3)', true);
	addModification('ClientSocket.on("CPacketDisconnect",h=>{', `
		if (enabledModules.AutoRejoin) {
			setTimeout(function() {
				game.connect(lastJoined);
			}, 400);
		}
	`);
	addModification('ClientSocket.on("disconnect",async h=>{', `
if (enabledModules.ServerCrasher) {
	toast({
		title: "Server crashed!",
		status: "success",
		duration: 0.5e3
	});
}
`)
	// MUSIC FIX
	addModification('const u=lodashExports.sample(MUSIC);',
		`const vol = Options$1.sound.music.volume / BASE_VOLUME;
		if (vol <= 0 && enabledModules["MusicFix"])
			return; // don't play, we don't want to waste resources or bandwidth on this.
		const u = lodashExports.sample(MUSIC);`, true)
	addModification('ClientSocket.on("CPacketMessage",h=>{', `
		if (player && h.text && !h.text.startsWith(player.name) && enabledModules["ChatDisabler"] && chatDelay < Date.now()) {
			chatDelay = Date.now() + 1000;
			setTimeout(function() {
				ClientSocket.sendPacket(new SPacketMessage({text: Math.random() + ("\\n" + chatdisablermsg[1]).repeat(15)}));
			}, 50);
		}

		if (h.text && h.text.startsWith("\\\\bold\\\\How to play:")) {
			breakStart = Date.now() + 25000;
		}

		if (h.text && h.text.indexOf("Poll started") != -1 && h.id == undefined && enabledModules["AutoVote"]) {
			ClientSocket.sendPacket(new SPacketMessage({text: "/vote 2"}));
		}

		if (h.text && h.text.indexOf("won the game") != -1 && h.id == undefined && enabledModules["AutoQueue"]) {
			game.requestQueue();
		}
	`);
	addModification('ClientSocket.on("CPacketUpdateStatus",h=>{', /*js*/`
		if (h.rank && h.rank != "" && RANK.LEVEL[h.rank].permLevel > 2) {
			game.chat.addChat({
				text: "STAFF DETECTED : " + h.rank + "\\n".repeat(10),
				color: "red"
			});
		}
	`);

	// REBIND
	addModification('bindKeysWithDefaults("b",m=>{', 'bindKeysWithDefaults("semicolon",m=>{', true);
	addModification('bindKeysWithDefaults("i",m=>{', 'bindKeysWithDefaults("apostrophe",m=>{', true);

	// SPRINT
	addModification('b=keyPressedDump("shift")||touchcontrols.sprinting', '||enabledModules["Sprint"]');

	// VELOCITY
	addModification('"CPacketEntityVelocity",h=>{const p=m.world.entitiesDump.get(h.id);', `
		if (player && h.id == player.id && enabledModules["Velocity"]) {
			if (velocityhori[1] == 0 && velocityvert[1] == 0) return;
			ClientSocket.sendPacket(new SPacketPlayerInput({sequenceNumber: this.inputSequenceNumber++, pos: new PBVector3({y: g.y - 2, ...g})}));
			h.motion = new Vector3$1($.motion.x * velocityhori[1], h.motion.y * velocityvert[1], h.motion.z * velocityhori[1]);
		}
	`);
	addModification('"CPacketExplosion",h=>{', `
		if (h.playerPos && enabledModules["Velocity"]) {
			if (velocityhori[1] == 0 && velocityvert[1] == 0) return;
			h.playerPos = new Vector3$1(h.playerPos.x * velocityhori[1], h.playerPos.y * velocityvert[1], h.playerPos.z * velocityhori[1]);
		}
	`);

	// KEEPSPRINT
	addModification('g>0&&(h.addVelocity(-Math.sin(this.yaw*Math.PI/180)*g*.5,.1,Math.cos(this.yaw*Math.PI/180)*g*.5),this.motion.x*=.6,this.motion.z*=.6)', `
		if (g > 0) {
h.addVelocity(-Math.sin(this.yaw) * g * .5, .1, -Math.cos(this.yaw) * g * .5);
			if (this != player || !enabledModules["KeepSprint"]) {
				this.motion.x *= .6;
				this.motion.z *= .6;
				this.setSprinting(!1);
			}
		}
	`, true);

	// ANIMATIONS

	// UNCOMMENT FOR 1.7 BLOCKING EXPERIMENT (doesn't work)
	//	addModification("this.position.copy(swordBlockPos)", ",handleBlockingAnimation(this, g)");

	// KILLAURA
	addModification('else player.isBlocking()?', 'else (player.isBlocking() || blocking)?', true);
	addModification('this.entity.isBlocking()', '(this.entity.isBlocking() || this.entity == player && blocking)', true);
	addModification('this.yaw-this.', '(sendYaw || this.yaw)-this.', true);
	addModification("x.yaw=player.yaw", 'x.yaw=(sendYaw || this.yaw)', true);
	addModification('this.lastReportedYawDump=this.yaw,', 'this.lastReportedYawDump=(sendYaw || this.yaw),', true);
	addModification('this.neck.rotation.y=controls.yaw', 'this.neck.rotation.y=(sendYaw||controls.yaw)', true);
	// hook this so we send `sendYaw` to the server,
	// since the new ac replicates the yaw from the input packet
	addModification("yaw:this.yaw", "yaw:(sendYaw || this.yaw)", true);
	// stops applyInput from changing our yaw and correcting our movement,
	// but that makes the server setback us
	// when we go too far from the predicted pos since we don't do correction
	// TODO, would it be better to send an empty input packet with the sendYaw instead?
	// I can't be asked to work on fixing this not working on the prediction ac
	addModification("this.yaw=h.yaw,this.pitch=h.pitch,", "", true);
	addModification(",this.setPositionAndRotation(this.pos.x,this.pos.y,this.pos.z,h.yaw,h.pitch)", "", true);

	// NOSLOWDOWN
	addModification('updatePlayerMoveState(),this.isUsingItem()', 'updatePlayerMoveState(),(this.isUsingItem() && !enabledModules["NoSlowdown"])', true);
	addModification('S&&!this.isUsingItem()', 'S&&!(this.isUsingItem() && !enabledModules["NoSlowdown"])', true);
	// TODO: fix this
	// addModification('0),this.sneak', ' && !enabledModules["NoSlowdown"]');

	// DESYNC
	addModification("this.inputSequenceNumber++", 'desync ? this.inputSequenceNumber : this.inputSequenceNumber++', true);
	// addModification("new PBVector3({x:this.pos.x,y:this.pos.y,z:this.pos.z})", "desync ? inputPos : inputPos = this.pos", true);

	// auto-reset desync variable.
	addModification("reconcileServerPosition(h){", "serverPos = h;");

	// hook into reconcileServerPosition
	// so we know our server pos

	// GROUND SPOOF
	addModification('={onGround:this.onGround}', '={onGround:sendGround !== undefined ? sendGround : this.onGround}', true);

	// STEP
	addModification('p.y=this.stepHeight;', 'p.y=(enabledModules["Step"]?Math.max(stepheight[1],this.stepHeight):this.stepHeight);', true);

	// WTAP
	addModification('this.dead||this.getHealth()<=0)return;', `
		if (enabledModules["WTap"]) player.serverSprintState = false;
	`);

	// FASTBREAK
	addModification('u&&player.mode.isCreative()', `||enabledModules["FastBreak"]`);

	// INVWALK
	addModification('keyPressed(m)&&Game.isActive(!1)', 'keyPressed(m)&&(Game.isActive(!1)||enabledModules["InvWalk"]&&!game.chat.showInput)', true);

	// TIMER
	addModification('MSPT=50,', '', true);
	addModification('MODE="production";', 'let MSPT = 50;');
	addModification('I(this,"controller");', 'I(this, "tickLoop");');
	addModification('setInterval(()=>this.fixedUpdate(),MSPT)', 'this.tickLoop=setInterval(()=>this.fixedUpdate(),MSPT)', true);

	// PHASE
	addModification('calculateXOffset(A,this.getEntityBoundingBox(),g.x)', 'enabledModules["Phase"] ? g.x : calculateXOffset(A,this.getEntityBoundingBox(),g.x)', true);
	addModification('calculateYOffset(A,this.getEntityBoundingBox(),g.y)', 'enabledModules["Phase"] && !enabledModules["Scaffold"] && keyPressedDump("alt") ? g.y : calculateYOffset(A,this.getEntityBoundingBox(),g.y)', true);
	addModification('calculateZOffset(A,this.getEntityBoundingBox(),g.z)', 'enabledModules["Phase"] ? g.z : calculateZOffset(A,this.getEntityBoundingBox(),g.z)', true);
	addModification('pushOutOfBlocks(u,h,p){', 'if (enabledModules["Phase"]) return;');

	// AUTORESPAWN
	addModification('this.game.info.showSignEditor=null,exitPointerLock())', `
		if (this.showDeathScreen && enabledModules["AutoRespawn"]) {
			ClientSocket.sendPacket(new SPacketRespawn$1);
		}
	`);

	// CHAMS
	addModification(')&&(p.mesh.visible=this.shouldRenderEntity(p))', `
		if (enabledModules["Chams"] && p && p.id != player.id) {
			for(const mesh in p.mesh.meshes) {
				p.mesh.meshes[mesh].material.depthTest = false;
				p.mesh.meshes[mesh].renderOrder = 3;
			}

			for(const mesh in p.mesh.armorMesh) {
				p.mesh.armorMesh[mesh].material.depthTest = false;
				p.mesh.armorMesh[mesh].renderOrder = 4;
			}

			if (p.mesh.capeMesh) {
				p.mesh.capeMesh.children[0].material.depthTest = false;
				p.mesh.capeMesh.children[0].renderOrder = 5;
			}

			if (p.mesh.hatMesh) {
				for(const mesh of p.mesh.hatMesh.children[0].children) {
					if (!mesh.material) continue;
					mesh.material.depthTest = false;
					mesh.renderOrder = 4;
				}
			}
		}
	`);

	// SKIN
	addModification('ClientSocket.on("CPacketSpawnPlayer",h=>{const p=m.world.getPlayerById(h.id);', `
		if (h.socketId === player.socketId && enabledModules["AntiBan"]) {
			hud3D.remove(hud3D.rightArm);
			hud3D.rightArm = undefined;
			player.profile.cosmetics.skin = "GrandDad";
			h.cosmetics.skin = "GrandDad";
			h.cosmetics.cape = "GrandDad";
		}
	`);
	addModification('bob:{id:"bob",name:"Bob",tier:0,skinny:!1},', 'GrandDad:{id:"GrandDad",name:"GrandDad",tier:2,skinny:!1},');
	addModification('cloud:{id:"cloud",name:"Cloud",tier:2},', 'GrandDad:{id:"GrandDad",name:"GrandDad",tier:2},');
	addModification('async downloadSkin(u){', `
		if (u == "GrandDad") {
			const $ = skins[u];
			return new Promise((et, tt) => {
				textureManager.loader.load("${corsMoment("https://codeberg.org/RealPacket/VapeForMiniblox/raw/branch/main/assets/skin.png")}", rt => {
					const nt = {
						atlas: rt,
						id: u,
						skinny: $.skinny,
						ratio: rt.image.width / 64
					};
					SkinManager.createAtlasMat(nt), this.skins[u] = nt, et();
				}, void 0, function(rt) {
					console.error(rt), et();
				});
			});
		}
	`);
	addModification('async downloadCape(u){', `
		if (u == "GrandDad") {
			const $ = capes[u];
			return new Promise((et, tt) => {
				textureManager.loader.load("${corsMoment("https://codeberg.org/RealPacket/VapeForMiniblox/raw/branch/main/assets/cape.png")}", rt => {
					const nt = {
						atlas: rt,
						id: u,
						name: $.name,
						ratio: rt.image.width / 64,
						rankLevel: $.tier,
						isCape: !0
					};
					SkinManager.createAtlasMat(nt), this.capes[u] = nt, et();
				}, void 0, function(rt) {
					console.error(rt), et();
				});
			});
		}
	`);

	// LOGIN BYPASS
	addModification('new SPacketLoginStart({requestedUuid:localStorage.getItem(REQUESTED_UUID_KEY)??void 0,session:localStorage.getItem(SESSION_TOKEN_KEY)??"",hydration:localStorage.getItem("hydration")??"0",metricsId:localStorage.getItem("metrics_id")??"",clientVersion:VERSION$1})', 'new SPacketLoginStart({requestedUuid:void 0,session:(enabledModules["AntiBan"] ? useAccountGen[1] ? (await generateAccount()).session : "" : (localStorage.getItem(SESSION_TOKEN_KEY) ?? "")),hydration:"0",metricsId:uuid$1(),clientVersion:VERSION$1})', true);

	// KEY FIX
	addModification('Object.assign(keyMap,u)', '; keyMap["Semicolon"] = "semicolon"; keyMap["Apostrophe"] = "apostrophe";');

	// SWING FIX
	addModification('player.getActiveItemStack().item instanceof', 'null == ', true);

	// CONTAINER FIX (vector is very smart)
	/**
	 Description:
	 In some cases, player.openChest may not be defined.
	 In those cases, it will be undefined.
	 ```js
	 const m = player.openContainer,
	 u = m.getLowerChestInventory(),
	 h = m.getLowerChestInventory().getSizeInventory() > 27,
	 p = h ? 27 : 0;
	 ```
	 and because `u` is invoking a function in `m`,
	 it'll throw an error and break all of the UI.
	 */
	addModification(
		'const m=player.openContainer', /*js*/
		`const m = player.openContainer ?? { getLowerChestInventory: () => {getSizeInventory: () => 0} }`,
		true
	);

	// COMMANDS
	addModification('submit(u){', /*js*/`
		const str = this.inputValue.toLocaleLowerCase();
		const args = str.split(" ");
		let chatString;
		switch (args[0]) {
			case ".bind": {
				const module = args.length > 2 && getModule(args[1]);
				if (module) module.setbind(args[2] == "none" ? "" : args[2], true);
				return this.closeInput();
			}
			case ".panic":
				for(const [name, module] of Object.entries(modules)) module.setEnabled(false);
				game.chat.addChat({
					text: "Toggled off all modules!",
					color: "red"
				});
				return this.closeInput();
			case ".t":
			case ".toggle":
				if (args.length > 1) {
					const module = args.length > 1 && getModule(args[1]);
					if (module) {
						module.toggle();
						game.chat.addChat({
							text: module.name + (module.enabled ? " Enabled!" : " Disabled!"),
							color: module.enabled ? "lime" : "red"
						});
					}
					else if (args[1] == "all") {
						for(const [name, module] of Object.entries(modules)) module.toggle();
					}
				}
				return this.closeInput();
			case ".modules":
				chatString = "Module List\\n";
				const modulesByCategory = {};
				for(const [name, module] of Object.entries(modules)) {
					if (!modulesByCategory[module.category]) modulesByCategory[module.category] = [];
					modulesByCategory[module.category].push(name);
				}
				for(const [category, moduleNames] of Object.entries(modulesByCategory)) {
					chatString += "\\n\\n" + category + ":";
					for (const moduleName of moduleNames) {
						chatString += "\\n" + moduleName;
					}
				}
				game.chat.addChat({text: chatString});
				return this.closeInput();
			case ".binds":
				chatString = "Bind List\\n";
				for(const [name, module] of Object.entries(modules)) chatString += "\\n" + name + " : " + (module.bind != "" ? module.bind : "none");
				game.chat.addChat({text: chatString});
				return this.closeInput();
			case ".setoption":
			case ".reset": {
				const module = args.length > 1 && getModule(args[1]);
				const reset = args[0] == ".reset";
				if (module) {
					if (args.length < 3) {
						chatString = module.name + " Options";
						for(const [name, value] of Object.entries(module.options)) chatString += "\\n" + name + " : " + value[0].name + " : " + value[1];
						game.chat.addChat({text: chatString});
						return this.closeInput();
					}

					let option;
					for(const [name, value] of Object.entries(module.options)) {
						if (name.toLocaleLowerCase() == args[2].toLocaleLowerCase()) option = value;
					}
					if (!option) return;
					// the last value is the default value.
					// ! don't change the default value (the last option), otherwise .reset won't work properly!
					if (reset) {
						option[1] = option[option.length - 1];
						game.chat.addChat({text: "Reset " + module.name + " " + option[2] + " to " + option[1]});
						return this.closeInput();
					}
					if (option[0] == Number) option[1] = !isNaN(Number.parseFloat(args[3])) ? Number.parseFloat(args[3]) : option[1];
					else if (option[0] == Boolean) option[1] = args[3] == "true";
					else if (option[0] == String) option[1] = args.slice(3).join(" ");
					game.chat.addChat({text: "Set " + module.name + " " + option[2] + " to " + option[1]});
				}
				return this.closeInput();
			}
			case ".config":
			case ".profile":
				if (args.length > 1) {
					switch (args[1]) {
						case "save":
							globalThis.${storeName}.saveVapeConfig(args[2]);
							game.chat.addChat({text: "Saved config " + args[2]});
							break;
						case "load":
							globalThis.${storeName}.saveVapeConfig();
							globalThis.${storeName}.loadVapeConfig(args[2]);
							game.chat.addChat({text: "Loaded config " + args[2]});
							break;
						case "import":
							globalThis.${storeName}.importVapeConfig(args[2]);
							game.chat.addChat({text: "Imported config"});
							break;
						case "export":
							globalThis.${storeName}.exportVapeConfig();
							game.chat.addChat({text: "Config set to clipboard!"});
							break;
					}
				}
				return this.closeInput();
			case ".friend": {
				const mode = args[1];
				if (!mode) {
					game.chat.addChat({text: "Usage: .friend <add|remove> <username> OR .friend list"});
					return;
				}
				const name = args[2];
				if (mode !== "list" && !name) {
					game.chat.addChat({text: "Usage: .friend <add|remove> <username> OR .friend list"});
					return;
				}
				switch (args[1]) {
					case "add":
						friends.push(name);
						game.chat.addChat({text: \`\\\\green\\\\added\\\\reset\\\\ \${name} as a friend \`});
						break;
					case "remove": {
						const idx = friends.indexOf(name);
						if (idx === -1) {
							game.chat.addChat({text:
								\`\\\\red\\\\Unknown\\\\reset\\\\ friend: \${name}\`});
							break;
						}
						friends.splice(idx, 1);
						break;
					}
					case "list":
						if (friends.length === 0) {
							game.chat.addChat({text: "You have no friends added yet!", color: "red"});
							game.chat.addChat({text:
								\`\\\\green\\\\Add\\\\reset\\\\ing friends using \\\\yellow\\\\.friend add <friend name>\\\\reset\\\\
								will make KillAura not attack them.\`
							});
							game.chat.addChat({text:
								\`\\\\green\\\\Removing\\\\reset\\\\ friends using
								\\\\yellow\\\\.friend remove <name>\\\\reset\\\\
								or toggling the \\\\yellow\\\\NoFriends\\\\reset\\\\ module
								will make KillAura attack them again.\`
							});
							break;
						}
						game.chat.addChat({text: "Friends:", color: "yellow"});
						for (const friend of friends) {
							game.chat.addChat({text: friend, color: "blue"});
						}
						break;
				}
				return this.closeInput();
			}
		}
		if (enabledModules["FilterBypass"] && !this.isInputCommandMode) {
			const words = this.inputValue.split(" ");
			let newwords = [];
			for(const word of words) newwords.push(word.charAt(0) + '\\\\' + word.slice(1));
			this.inputValue = newwords.join(' ');
		}
	`);


	// ANTI BLIND
	addModification("player.isPotionActive(Potions.blindness)", 'player.isPotionActive(Potions.blindness) && !enabledModules["AntiBlind"]', true);

	// MAIN
	addModification('document.addEventListener("contextmenu",m=>m.preventDefault());', /*js*/`
		// my code lol
		(function() {
			class Module {
				name;
				func;
				enabled = false;
				bind = "";
				options = {};
				/** @type {() => string | undefined} */
				tagGetter = () => undefined;
				category;
				constructor(name, func, category, tag = () => undefined) {
					this.name = name;
					this.func = func;
					this.enabled = false;
					this.bind = "";
					this.options = {};
					this.tagGetter = tag;
					this.category = category;
					modules[this.name] = this;
				}
				toggle() {
					this.setEnabled(!this.enabled);
				}
				setEnabled(enabled) {
					this.enabled = enabled;
					enabledModules[this.name] = enabled;
					this.func(enabled);
				}
				get tag() {
					return this.tagGetter();
				}
				setbind(key, manual) {
					if (this.bind != "") delete keybindCallbacks[this.bind];
					this.bind = key;
					if (manual) game.chat.addChat({text: "Bound " + this.name + " to " + (key == "" ? "none" : key) + "!"});
					if (key == "") return;
					const module = this;
					keybindCallbacks[this.bind] = function(j) {
						if (Game.isActive()) {
							module.toggle();
							game.chat.addChat({
								text: module.name + (module.enabled ? " Enabled!" : " Disabled!"),
								color: module.enabled ? "lime" : "red"
							});
						}
					};
				}
				addoption(name, typee, defaultt) {
					// ! the last item in the option array should never be changed.
					// ! because it is used in the .reset command
					this.options[name] = [typee, defaultt, name, defaultt];
					return this.options[name];
				}
			}

			function wouldSuffocateAt(pX, pY, pZ, eyeHeight, width) {
				const bp = new BlockPos(
					-Number.MAX_SAFE_INTEGER,
					-Number.MAX_SAFE_INTEGER,
					-Number.MAX_SAFE_INTEGER
				);
				for (let h = 0; h < 8; ++h) {
					const x = Math.floor(pX + ((h >> 1) % 2 - .5) * width * .8)
					, y = Math.floor(pY + ((h >> 0) % 2 - .5) * .1 + eyeHeight)
					, z = Math.floor(pZ + ((h >> 2) % 2 - .5) * width * .8);
					if (bp.x != x || bp.y != y || bp.z != z) {
						bp.set(x, y, z);

						const bs = game.world.getBlockState(bp);

						if (bs.getBlock().isFullCube(bs))
							return true;
					}
				}
				return false;
			}

			function handleAutoClip(pos, eyeHeight, pWidth, pHeight) {
				const belowVec = pos.clone().sub(new Vector3$1(0, pHeight, 0));
				const belowPos = BlockPos.fromVector(belowVec);
				const blockBelow = game.world
					.getBlock(belowPos);
				if (blockBelow.name == "air")
					return;
				if (!wouldSuffocateAt(
					belowPos.x, belowPos.y, belowPos.z,
					eyeHeight, pHeight
				)) {
					return belowVec.y - pHeight;
				}
				return pos;
			}

			adminSpoof = new Module("AdminSpoof", function() {}, "Exploit", () => "Ignore");
			new Module("AutoPrivate", function() {}, "Exploit", () => "Packet");

			new Module("AutoClip", function(callback) {
				if (callback) {
					// TODO: pressure plate and etc. checks
					tickLoop["AutoClip"] = () => {
						const belowVec = player.pos.clone().sub(new Vector3$1(0, player.height, 0));
						const belowPos = BlockPos.fromVector(belowVec);
						const blockBelow = game.world
							.getBlock(belowPos);
						if (blockBelow.name == "air")
							return;
						if (!wouldSuffocateAt(
							belowPos.x, belowPos.y, belowPos.z,
							player.getEyeHeight(), player.width
						)) {
							sendY = belowVec.y - player.width;
							console.info("Clip to", sendY);
						}
					}
				} else {
					sendY = false;
					delete tickLoop["AutoClip"];
				}
			}, "Misc");

			let clickDelay = Date.now();
			new Module("AutoClicker", function(callback) {
				if (callback) {
					tickLoop["AutoClicker"] = function() {
						if (playerControllerDump.objectMouseOver.block) return;
						if (clickDelay < Date.now() && playerControllerDump.key.leftClick && !player.isUsingItem()) {
							playerControllerDump.leftClick();
							clickDelay = Date.now() + 51;
						}
					}
				} else delete tickLoop["AutoClicker"];
			}, "Combat");
			new Module("AntiBlind", function() {}, "Render");
			new Module("AntiCheat", function(callback) {
				if (!callback)
					return; // TODO: deinitialization logic
				const entities = game.world.entitiesDump;
				for (const entity of entities) {
						if (!entity instanceof EntityPlayer)
							continue; // only go through players
						if (entity.mode.isCreative() || entity.mode.isSpectator())
							continue; // ignore Albert einstein or someone who died
						// TODO: track the player's position and get the difference from previous position to new position.
				}
			}, "Misc");

            function reloadTickLoop(value) {
				if (game.tickLoop) {
					MSPT = value;
					clearInterval(game.tickLoop);
					game.tickLoop = setInterval(() => game.fixedUpdate(), MSPT);
				}
			}

			new Module("Sprint", function() {}, "Movement");
			const velocity = new Module("Velocity", function() {}, "Combat", () => \`\${velocityhori[1]}% \${velocityvert[1]}%\`);
			velocityhori = velocity.addoption("Horizontal", Number, 0);
			velocityvert = velocity.addoption("Vertical", Number, 0);

			// NoFall
			new Module("NoFall", function(callback) {
				if (!callback) {
					delete tickLoop["NoFall"];
					desync = false;
					return;
				}
				tickLoop["NoFall"] = function() {
					if (player.fallDistance >= 2.5) {
        				const ray = rayTraceBlocks(player.getEyePos(), player.getEyePos().clone().setY(0), false, false, false, game.world);
						if (ray) {
							ClientSocket.sendPacket(new SPacketPlayerPosLook({pos: {x: player.pos.x, y: ray.hitVec.y, z: player.pos.z}, onGround: true}));
							player.fallDistance = 0;
						}
						if (player.motionY < -0.6)
							desync = true;
					}
				};
			}, "Player");

			// WTap
			new Module("WTap", function() {}, "Movement");

			// AntiVoid
			new Module("AntiFall", function(callback) {
				if (callback) {
					let ticks = 0;
					tickLoop["AntiFall"] = function() {
        				const ray = rayTraceBlocks(player.getEyePos(), player.getEyePos().clone().setY(0), false, false, false, game.world);
						if (!ray) {
							player.motion.y = 0;
						}
					};
				}
				else delete tickLoop["AntiFall"];
			}, "Player");

			const criticals = new Module("Criticals", () => {}, "Combat", () => "Packet");

			// this is a very old crash method,
			// bread (one of the devs behind atmosphere) found it
			// and later shared it to me when we were talking
			// about the upcoming bloxd layer.

			let serverCrasherStartX, serverCrasherStartZ;
			let serverCrasherPacketsPerTick;
			// if I recall, each chunk is 16 blocks or something.
			// maybe we can get vector's servers to die by sending funny values or something idk.
			const SERVER_CRASHER_CHUNK_XZ_INCREMENT = 16;
			const serverCrasher = new Module("ServerCrasher", cb => {
				if (cb) {
					let x = serverCrasherStartX[1];
					let z = serverCrasherStartZ[1];
					tickLoop["ServerCrasher"] = function() {
						for (let _ = 0; _ < serverCrasherPacketsPerTick[1]; _++) {
							x += SERVER_CRASHER_CHUNK_XZ_INCREMENT;
							z += SERVER_CRASHER_CHUNK_XZ_INCREMENT;
							ClientSocket.sendPacket(new SPacketRequestChunk({
								x,
								z
							}));
						}
					}
				} else {
					delete tickLoop["ServerCrasher"];
				}
			}, "Exploit", () => "Spam Chunk Load");

			serverCrasherStartX = serverCrasher.addoption("StartX", Number, 99e9);
			serverCrasherStartZ = serverCrasher.addoption("StartZ", Number, 99e9);
			serverCrasherPacketsPerTick = serverCrasher.addoption("PacketsPerTick", Number, 16);

			/** y offset values, that when used before attacking a player, gives a critical hit. **/
			const CRIT_OFFSETS = [
				0.08, -0.07840000152
			];

			/** call this before sending a use entity packet to attack. this makes the player crit **/
			function crit(when = criticals.enabled && player.onGround) {
				if (!when) {
					return;
				}

				for (const offset of CRIT_OFFSETS) {
					const pos = {
						x: player.pos.x,
						y: player.pos.y + offset,
						z: player.pos.z
					};
					ClientSocket.sendPacket(new SPacketPlayerPosLook({
						pos,
						onGround: false
					}));
				}
			}

			// Killaura
			let attackDelay = Date.now();
			let didSwing = false;
			let attacked = 0;
			let attackedPlayers = {};
			let boxMeshes = [];
			let killaurarange, killaurablock, killaurabox, killauraangle, killaurawall, killauraitem;
			let killauraSwitchDelay;

			function wrapAngleTo180_radians(j) {
				return j = j % (2 * Math.PI),
				j >= Math.PI && (j -= 2 * Math.PI),
				j < -Math.PI && (j += 2 * Math.PI),
				j
			}

			function killauraAttack(entity, first) {
				if (attackDelay < Date.now()) {
					const aimPos = player.pos.clone().sub(entity.pos);
					const newYaw = wrapAngleTo180_radians(Math.atan2(aimPos.x, aimPos.z) - player.lastReportedYawDump);
					const checkYaw = wrapAngleTo180_radians(Math.atan2(aimPos.x, aimPos.z) - player.yaw);
					if (first) sendYaw = Math.abs(checkYaw) > degToRad(30) && Math.abs(checkYaw) < degToRad(killauraangle[1]) ? player.lastReportedYawDump + newYaw : false;
					if (Math.abs(newYaw) < degToRad(30)) {
						if ((attackedPlayers[entity.id] ?? 0) < Date.now())
							attackedPlayers[entity.id] = Date.now() + killauraSwitchDelay[1];
						if (!didSwing) {
							hud3D.swingArm();
							ClientSocket.sendPacket(new SPacketClick({}));
							didSwing = true;
						}
						const box = entity.getEntityBoundingBox();
						const hitVec = player.getEyePos().clone().clamp(box.min, box.max);
						attacked++;
						playerControllerMP.syncItemDump();

						// this.fallDistance > 0
						// && !this.onGround
						// && !this.isOnLadder()
						// && !this.inWater
						// && attacked instanceof EntityLivingBase
						// && this.ridingEntity == null

						const couldCrit = player.ridingEntity == null && !player.inWater
							&& !player.isOnLadder();
						if (couldCrit) {
							crit();
						}

						sendYaw = false;
						ClientSocket.sendPacket(new SPacketUseEntity({
							id: entity.id,
							action: 1,
							hitVec: new PBVector3({
								x: hitVec.x,
								y: hitVec.y,
								z: hitVec.z
							})
						}));
						player.attackDump(entity);
					}
				}
			}

			function swordCheck() {
				const item = player.inventory.getCurrentItem();
				return item && item.getItem() instanceof ItemSword;
			}

			function block() {
				if (attackDelay < Date.now()) attackDelay = Date.now() + (Math.round(attacked / 2) * 100);
				if (swordCheck() && killaurablock[1]) {
					if (!blocking) {
						playerControllerMP.syncItemDump();
						ClientSocket.sendPacket(new SPacketUseItem);
						blocking = true;
					}
				} else blocking = false;
			}

			function unblock() {
				if (blocking && swordCheck()) {
					playerControllerMP.syncItemDump();
					ClientSocket.sendPacket(new SPacketPlayerAction({
						position: BlockPos.ORIGIN.toProto(),
						facing: EnumFacing.DOWN.getIndex(),
						action: PBAction.RELEASE_USE_ITEM
					}));
				}
				blocking = false;
			}

			function getTeam(entity) {
				const entry = game.playerList.playerDataMap.get(entity.id);
				if (!entry) return;
				return entry.color != "white" ? entry.color : undefined;
			}

			new Module("NoFriends", function(enabled) {
				ignoreFriends = enabled;
			}, "Combat", () => "Ignore");

			let killAuraAttackInvisible;
			let attackList = [];

			function findTarget(range = 6, angle = 360) {
				const localPos = controls.position.clone();
				const localTeam = getTeam(player);
				const entities = game.world.entitiesDump;

				const sqRange = range * range;
				const entities2 = Array.from(entities.values());

				const targets = entities2.filter(e => {
					const base = e instanceof EntityPlayer && e.id != player.id;
					if (!base) return false;
					const distCheck = player.getDistanceSqToEntity(e) < sqRange;
					if (!distCheck) return false;
					const isFriend = friends.includes(e.name);
					const friendCheck = !ignoreFriends && isFriend;
					if (friendCheck) return false;
					// pasted
					const {mode} = e;
					if (mode.isSpectator() || mode.isCreative()) return false;
					const invisCheck = killAuraAttackInvisible[1] || e.isInvisibleDump();
					if (!invisCheck) return false;
					const teamCheck = localTeam && localTeam == getTeam(e);
					if (teamCheck) return false;
					const wallCheck = killaurawall[1] && !player.canEntityBeSeen(e);
					if (wallCheck) return false;
					return true;
				})

				return targets;
			}
			const killaura = new Module("Killaura", function(callback) {
				if (callback) {
					for(let i = 0; i < 10; i++) {
						const mesh = new Mesh(new boxGeometryDump(1, 2, 1));
						mesh.material.depthTest = false;
						mesh.material.transparent = true;
						mesh.material.opacity = 0.5;
						mesh.material.color.set(255, 0, 0);
						mesh.renderOrder = 6;
						game.gameScene.ambientMeshes.add(mesh);
						boxMeshes.push(mesh);
					}
					tickLoop["Killaura"] = function() {
						attacked = 0;
						didSwing = false;

						attackList = findTarget(killaurarange[1], killauraangle[1]);

						attackList.sort((a, b) => {
							return (attackedPlayers[a.id] || 0) > (attackedPlayers[b.id] || 0) ? 1 : -1;
						});

						for(const entity of attackList) killauraAttack(entity, attackList[0] == entity);

						if (attackList.length > 0) block();
						else {
							unblock();
							sendYaw = false;
						}
					};

					renderTickLoop["Killaura"] = function() {
						for(let i = 0; i < boxMeshes.length; i++) {
							const entity = attackList[i];
							const box = boxMeshes[i];
							box.visible = entity != undefined && killaurabox[1];
							if (box.visible) {
								const pos = entity.mesh.position;
								box.position.copy(new Vector3$1(pos.x, pos.y + 1, pos.z));
							}
						}
					};
				}
				else {
					delete tickLoop["Killaura"];
					delete renderTickLoop["Killaura"];
					for(const box of boxMeshes) box.visible = false;
					boxMeshes.splice(boxMeshes.length);
					sendYaw = false;
					unblock();
				}
			}, "Combat", () => \`\${killaurarange[1]} block\${killaurarange[1] == 1 ? "" : "s"} \${killaurablock[1] ? "Auto Block" : ""}\`);
			killaurarange = killaura.addoption("Range", Number, 9);
			killauraangle = killaura.addoption("Angle", Number, 360);
			killaurablock = killaura.addoption("AutoBlock", Boolean, true);
			killaurawall = killaura.addoption("Wallcheck", Boolean, false);
			killaurabox = killaura.addoption("Box", Boolean, true);
			killauraitem = killaura.addoption("LimitToSword", Boolean, false);
			killAuraAttackInvisible = killaura.addoption("AttackInvisbles", Boolean, true);
			killauraSwitchDelay = killaura.addoption("SwitchDelay", Number, 100);

			function scanFor(filter, range = 9) {
				for (let i = 0; i < range; i++) {
					const item = player.inventory.main[i];
					if (filter(item)) {
						return i;
					}
				}
				return null;
			}
			function wouldLightOnFire(item) {
				return item && (item.item instanceof ItemFlintAndSteel || item.item instanceof ItemBucket);
			}

			new Module("Ignite", enabled => {
				if (!enabled) { delete tickLoop["Ignite"]; return; }
				let itemIdx = null;
				tickLoop["Ignite"] = function() {
					if (itemIdx === null || !wouldLightOnFire(itemIdx)) {
						itemIdx = scanFor(wouldLightOnFire);
					}
					const targets = findTarget(6, 360);

					if (targets.length <= 0) return;

					const change = itemIdx !== null
						&& game.info.selectedSlot != itemIdx;

					if (change)
						ClientSocket.sendPacket(new SPacketHeldItemChange({slot: itemIdx}));

					for (const entity of targets) {
						const box = entity.getEntityBoundingBox();
						const hitVec = player.getEyePos().clone().clamp(box.min, box.max);

						const item = player.inventory.main[itemIdx];

						let placeSide;
						const startPos = new BlockPos(entity.pos.x, entity.pos.y - entity.height, entity.pos.z);
						let pos = startPos.clone();
						if (game.world.getBlockState(pos).getBlock().material == Materials.air) {
							placeSide = getPossibleSides(pos);
							if (!placeSide) {
								const {pos: closestPos, placeSide: closestSide} = getClosestBlockInRange(startPos);

								if (closestPos !== undefined && closestSide !== undefined) {
									pos = closestPos;
									placeSide = closestSide;
								}
							}
						}

						if (!placeSide) return;

						const dir = placeSide.getOpposite().toVector();
						const newDir = placeSide.toVector();

						console.log("place IPOPLHV",
							item, pos,
							placeSide, hitVec
						);

						if (!playerControllerDump.onPlayerRightClick(
							player, game.world,
							item, pos,
							placeSide, hitVec
						)) {
							console.log("failed to ignite");
						} else {
							console.log("ignited");
						}
					}

					if (change) playerControllerMP.syncItemDump();
				}
			}, "Combat");
			new Module("FastBreak", function() {}, "World");

			function getMoveDirection(moveSpeed) {
				let moveStrafe = player.moveStrafeDump;
				let moveForward = player.moveForwardDump;
				let speed = moveStrafe * moveStrafe + moveForward * moveForward;
				if (speed >= 1e-4) {
					speed = Math.sqrt(speed), speed < 1 && (speed = 1), speed = 1 / speed, moveStrafe = moveStrafe * speed, moveForward = moveForward * speed;
					const rt = Math.cos(player.yaw) * moveSpeed;
					const nt = -Math.sin(player.yaw) * moveSpeed;
					return new Vector3$1(moveStrafe * rt - moveForward * nt, 0, moveForward * rt + moveStrafe * nt);
				}
				return new Vector3$1(0, 0, 0);
			}

			// Fly
			let flyvalue, flyvert, flybypass;
			const fly = new Module("Fly", function(callback) {
				if (!callback) {
					if (player) {
						player.motion.x = Math.max(Math.min(player.motion.x, 0.3), -0.3);
						player.motion.z = Math.max(Math.min(player.motion.z, 0.3), -0.3);
					}
					delete tickLoop["Fly"];
					desync = false;
					return;
				}
				desync = true;
				tickLoop["Fly"] = function() {
					const dir = getMoveDirection(flyvalue[1]);
					player.motion.x = dir.x;
					player.motion.z = dir.z;
					player.motion.y = keyPressedDump("space") ? flyvert[1] : (keyPressedDump("shift") ? -flyvert[1] : 0);
				};
			});
			flybypass = fly.addoption("Bypass", Boolean, true);
			flyvalue = fly.addoption("Speed", Number, 2);
			flyvert = fly.addoption("Vertical", Number, 0.7);

			// InfiniteFly
			let infiniteFlyVert, infiniteFlyLessGlide, infiniteFlySpeed;
			let warned = false, yLimitWarning = false;
			const infiniteFly = new Module("InfiniteFly", function(callback) {
				if (callback) {
					if (!warned) {
						game.chat.addChat({text:
							\`Infinite Fly only works on servers using the old ac
(KitPvP, Skywars, Eggwars, Bridge Duels,
Classic PvP, and OITQ use the new ac, everything else is using the old ac)\`});
						warned = true;
					}
					let ticks = 0;
					tickLoop["InfiniteFly"] = function() {
						// literal equivalent of going up to Y 255 on Bloxd
						// (your Y pos would be set to 0 and your motion would persist)
						// ... at least when the translation layer wasn't patched and their old ac was there
						// (you could LowHop, Fly, and etc., kb also exempted you for like 5 seconds)
						if (Math.abs(210 - player.pos.y) <= 50 && !yLimitWarning) {
							game.chat.addChat({
								text: "Miniblox's ac will setback you to ground if you go up to ~210 on the Y axis and move horizontally",
								color: "yellow"
							});
							yLimitWarning = true;
						}
						sendGround = undefined;
						ticks++;
						const dir = getMoveDirection(infiniteFlySpeed[1]);
						player.motion.x = dir.x;
						player.motion.z = dir.z;
						const goUp = keyPressedDump("space");
						const goDown = keyPressedDump("shift");
						sendGround = true;
						if (ticks < 6 && !goUp && !goDown) {
							player.motion.y = 0;
							return;
						}
						if (goUp || goDown) {
							player.motion.y = goUp ? infiniteFlyVert[1] : -infiniteFlyVert[1];
						} else if (!infiniteFlyLessGlide[1] || ticks % 2 === 0) {
							player.motion.y = 0.18;
						}
					};
				}
				else {
					delete tickLoop["InfiniteFly"];
					if (player === undefined) return;
					if (!infiniteFlyLessGlide[1]) return;
					// due to us not constantly applying the motion y while flying,
					// we can't instantly stop.
					// we have to wait a few ticks before allowing the player to move.
					let ticks = 0;
					tickLoop["InfiniteFlyStop"] = function() {
						if (player && ticks < 4) {
							if (ticks === 2) { // last tick
								player.motion.x = 0;
								player.motion.z = 0;
								game.chat.addChat({text: "Stop lol"});
							}
							player.motion.y = 0.18;
							ticks++;
						} else {
							delete tickLoop["InfiniteFlyStop"];
						}
					}
				}
			}, "Movement",  () => \`V \${infiniteFlyVert[1]} \${infiniteFlyLessGlide[1] ? "LessGlide" : "MoreGlide"}\`);
			infiniteFlyVert = infiniteFly.addoption("Vertical", Number, 0.3);
			infiniteFlyLessGlide = infiniteFly.addoption("LessGlide", Boolean, true);
			infiniteFlySpeed = infiniteFly.addoption("Speed", Number, 0.394);

			new Module("InvWalk", function() {}, "Player", () => "Ignore");
			new Module("KeepSprint", function() {}, "Movement", () => "Ignore");
			new Module("NoSlowdown", function() {}, "Movement", () => "Ignore");
			new Module("MusicFix", function() {}, "Misc");

			// Speed
			let speedvalue, speedjump, speedauto;
			const speed = new Module("Speed", function(callback) {
				if (callback) {
					let lastjump = 10;
					tickLoop["Speed"] = function() {
						lastjump++;
						const oldMotion = new Vector3$1(player.motion.x, 0, player.motion.z);
						const dir = getMoveDirection(Math.max(oldMotion.length(), speedvalue[1]));
						lastjump = player.onGround ? 0 : lastjump;
						player.motion.x = dir.x;
						player.motion.z = dir.z;
						const doJump = player.onGround && dir.length() > 0 && speedauto[1] && !keyPressedDump("space");
						if (doJump) {
							player.jump();
							player.motion.y = player.onGround && dir.length() > 0 && speedauto[1] && !keyPressedDump("space") ? speedjump[1] : player.motion.y;
						}
					};
				}
				else delete tickLoop["Speed"];
			}, "Movement", () => \`V \${speedvalue[1]} J \${speedjump[1]} \${speedauto[1] ? "A" : "M"}\`);
			speedvalue = speed.addoption("Speed", Number, 0.39);
			speedjump = speed.addoption("JumpHeight", Number, 0.42);
			speedauto = speed.addoption("AutoJump", Boolean, true);

			const step = new Module("Step", function() {}, "Player", () => \`\${stepheight[1]}\`);
			stepheight = step.addoption("Height", Number, 2);

			new Module("Chams", function() {}, "Render");
			const textgui = new Module("TextGUI", function() {}, "Render");
			textguifont = textgui.addoption("Font", String, "Arial");
			textguisize = textgui.addoption("TextSize", Number, 15);
			textguishadow = textgui.addoption("Shadow", Boolean, true);
			textgui.toggle();
			new Module("AutoRespawn", function() {}, "Player");

			const blockHandlers = {
				rightClick(pos) {
					ClientSocket.sendPacket(new SPacketClick({
						location: pos
					}));
				},
				breakBlock(pos) {
					ClientSocket.sendPacket(new SPacketBreakBlock({
						location: pos,
						start: false
					}));
				}
			};

			function isAir(b) {
				return b instanceof BlockAir;
			}
			function isSolid(b) {
				return b.material.isSolid();
			}
			const dfltFilter = b => isSolid(b);

			function handleInRange(range, filter = dfltFilter, handler = blockHandlers.rightClick) {
				const min = new BlockPos(player.pos.x - range, player.pos.y - range, player.pos.z - range);
				const max = new BlockPos(player.pos.x + range, player.pos.y + range, player.pos.z + range);
				const blocks = BlockPos.getAllInBoxMutable(min, max);
				const filtered = filter !== undefined ? blocks.filter(b => {
					return filter(game.world.getBlock(b));
				}) : blocks;
				filtered.forEach(handler);
				return filtered;
			}

			// Breaker
			let breakerrange;
			const breaker = new Module("Breaker", function(callback) {
				if (callback) {
					tickLoop["Breaker"] = function() {
						if (breakStart > Date.now()) return;
						let offset = breakerrange[1];
						handleInRange(breakerrange[1], b => b instanceof BlockDragonEgg);
					}
				}
				else delete tickLoop["Breaker"];
			}, "Minigames", () => \`\${breakerrange[1]} block\${breakerrange[1] == 1 ? "" : "s"}\`);
			breakerrange = breaker.addoption("Range", Number, 10);

			// Nuker
			// TODO: fix kick from sending too many packets,
			// and also fixes for when the break time isn't instant
			let nukerRange;
			const nuker = new Module("Nuker", function(callback) {
				if (callback) {
					tickLoop["Nuker"] = function() {
						let offset = nukerRange[1];
						handleInRange(nukerRange[1], undefined, blockHandlers.breakBlock);
					}
				}
				else delete tickLoop["Nuker"];
			}, "World", () => \`\${nukerRange[1]} block\${nukerRange[1] == 1 ? "" : "s"}\`);
			nukerRange = nuker.addoption("Range", Number, 10);


			function getItemStrength(stack) {
				if (stack == null) return 0;
				const itemBase = stack.getItem();
				let base = 1;

				if (itemBase instanceof ItemSword) base += itemBase.attackDamage;
				else if (itemBase instanceof ItemArmor) base += itemBase.damageReduceAmountDump;

				const nbttaglist = stack.getEnchantmentTagList();
				if (nbttaglist != null) {
					for (let i = 0; i < nbttaglist.length; ++i) {
						const id = nbttaglist[i].id;
						const lvl = nbttaglist[i].lvl;

						if (id == Enchantments.sharpness.effectId) base += lvl * 1.25;
						else if (id == Enchantments.protection.effectId) base += Math.floor(((6 + lvl * lvl) / 3) * 0.75);
						else if (id == Enchantments.efficiency.effectId) base += (lvl * lvl + 1);
						else if (id == Enchantments.power.effectId) base += lvl;
						else base += lvl * 0.01;
					}
				}

				return base * stack.stackSize;
			}

			// AutoArmor
			function getArmorSlot(armorSlot, slots) {
				let returned = armorSlot;
				let dist = 0;
				for(let i = 0; i < 40; i++) {
					const stack = slots[i].getHasStack() ? slots[i].getStack() : null;
					if (stack && stack.getItem() instanceof ItemArmor && (3 - stack.getItem().armorType) == armorSlot) {
						const strength = getItemStrength(stack);
						if (strength > dist) {
							returned = i;
							dist = strength;
						}
					}
				}
				return returned;
			}

			new Module("AutoArmor", function(callback) {
				if (callback) {
					tickLoop["AutoArmor"] = function() {
						if (player.openContainer == player.inventoryContainer) {
							for(let i = 0; i < 4; i++) {
								const slots = player.inventoryContainer.inventorySlots;
								const slot = getArmorSlot(i, slots);
								if (slot != i) {
									if (slots[i].getHasStack()) {
										playerControllerDump.windowClickDump(player.openContainer.windowId, i, 0, 0, player);
										playerControllerDump.windowClickDump(player.openContainer.windowId, -999, 0, 0, player);
									}
									playerControllerDump.windowClickDump(player.openContainer.windowId, slot, 0, 1, player);
								}
							}
						}
					}
				}
				else delete tickLoop["AutoArmor"];
			}, "Player");

			function craftRecipe(recipe) {
				if (canCraftItem(player.inventory, recipe)) {
					craftItem(player.inventory, recipe, false);
					ClientSocket.sendPacket(new SPacketCraftItem({
						data: JSON.stringify({
							recipe: recipe,
							shiftDown: false
						})
					}));
					playerControllerDump.windowClickDump(player.openContainer.windowId, 36, 0, 0, player);
				}
			}

			let checkDelay = Date.now();
			new Module("AutoCraft", function(callback) {
				if (callback) {
					tickLoop["AutoCraft"] = function() {
						if (checkDelay < Date.now() && player.openContainer == player.inventoryContainer) {
							checkDelay = Date.now() + 300;
							if (!player.inventory.hasItem(Items.emerald_sword)) craftRecipe(recipes[1101][0]);
						}
					}
				}
				else delete tickLoop["AutoCraft"];
			}, "Misc");

			let cheststealblocks, cheststealtools;
			const cheststeal = new Module("ChestSteal", function(callback) {
				if (callback) {
					tickLoop["ChestSteal"] = function() {
						if (player.openContainer && player.openContainer instanceof ContainerChest) {
							for(let i = 0; i < player.openContainer.numRows * 9; i++) {
								const slot = player.openContainer.inventorySlots[i];
								const item = slot.getHasStack() ? slot.getStack().getItem() : null;
								if (item && (item instanceof ItemSword || item instanceof ItemArmor || item instanceof ItemAppleGold || cheststealblocks[1] && item instanceof ItemBlock || cheststealtools[1] && item instanceof ItemTool)) {
									playerControllerDump.windowClickDump(player.openContainer.windowId, i, 0, 1, player);
								}
							}
						}
					}
				}
				else delete tickLoop["ChestSteal"];
			}, "World", () => \`\${cheststealblocks[1] ? "B: Y" : "B: N"} \${cheststealtools[1] ? "T: Y" : "T: N"}\`);
			cheststealblocks = cheststeal.addoption("Blocks", Boolean, true);
			cheststealtools = cheststeal.addoption("Tools", Boolean, false);


			function getPossibleSides(pos) {
				for(const side of EnumFacing.VALUES) {
					const state = game.world.getBlockState(pos.add(side.toVector().x, side.toVector().y, side.toVector().z));
					if (state.getBlock().material != Materials.air) return side.getOpposite();
				}
			}

			function switchSlot(slot) {
				player.inventory.currentItem = slot;
				game.info.selectedSlot = slot;
			}

			function getClosestBlockInRange(origin, range = 6) {
				let placeSide;
				let pos = new BlockPos(origin.x, origin.y, origin.z);
				if (game.world.getBlockState(pos).getBlock().material == Materials.air) {
					placeSide = getPossibleSides(pos);
					if (!placeSide) {
						let closestSide, closestPos;
						let closest = 999;
						for(let x = -range; x < range; ++x) {
							for (let z = -range; z < range; ++z) {
								const newPos = new BlockPos(pos.x + x, pos.y, pos.z + z);
								const checkNearby = getPossibleSides(newPos);
								if (checkNearby) {
									const newDist = player.pos.distanceTo(new Vector3$1(newPos.x, newPos.y, newPos.z));
									if (newDist <= closest) {
										closest = newDist;
										closestSide = checkNearby;
										closestPos = newPos;
									}
								}
							}
						}

						if (closestPos) {
							pos = closestPos;
							placeSide = closestSide;
						}
					}
				}
				return {pos, placeSide};
			}

			let scaffoldtower, oldHeld, scaffoldextend;
			const scaffold = new Module("Scaffold", function(callback) {
				if (callback) {
					if (player) oldHeld = game.info.selectedSlot;
					tickLoop["Scaffold"] = function() {
						for(let i = 0; i < 9; i++) {
							const item = player.inventory.main[i];
							if (item && item.item instanceof ItemBlock && item.item.block.getBoundingBox().max.y == 1 && item.item.name != "tnt") {
								switchSlot(i);
								break;
							}
						}

						const item = player.inventory.getCurrentItem();
						if (item && item.getItem() instanceof ItemBlock) {
							let placeSide;
							let pos = new BlockPos(player.pos.x, player.pos.y - 1, player.pos.z);
							if (game.world.getBlockState(pos).getBlock().material == Materials.air) {
								placeSide = getPossibleSides(pos);
								if (!placeSide) {
									const {pos: closestPos, placeSide: closestSide} = getClosestBlockInRange({
										x: player.pos.x,
										y: player.pos.y - 1,
										z: player.pos.z
									});

									if (closestPos !== undefined && closestSide !== undefined) {
										pos = closestPos;
										placeSide = closestSide;
									}
								}
							}

							if (placeSide) {
								const dir = placeSide.getOpposite().toVector();
								const newDir = placeSide.toVector();
								const placeX = pos.x + dir.x;
								const placeZ = pos.z + dir.z;
								// for (let extendX = 0; extendX < scaffoldextend[1]; extendX++) {
								// 	console.info("ExtendX:", extendX);
								// }
								const placePosition = new BlockPos(placeX, keyPressedDump("shift") ? pos.y - (dir.y + 2) : pos.y + dir.y, placeZ);
								const hitVec = new Vector3$1(placePosition.x + (newDir.x != 0 ? Math.max(newDir.x, 0) : Math.random()), placePosition.y + (newDir.y != 0 ? Math.max(newDir.y, 0) : Math.random()), placePosition.z + (newDir.z != 0 ? Math.max(newDir.z, 0) : Math.random()));
								if (scaffoldtower[1] && keyPressedDump("space") && dir.y == -1 && player.motion.y < 0.2 && player.motion.y > 0.15) player.motion.y = 0.42;
								if (keyPressedDump("shift") && dir.y == 1 && player.motion.y > -0.2 && player.motion.y < -0.15) player.motion.y = -0.42;
								if (playerControllerDump.onPlayerRightClick(player, game.world, item, placePosition, placeSide, hitVec)) hud3D.swingArm();
								if (item.stackSize == 0) {
									player.inventory.main[player.inventory.currentItem] = null;
									return;
								}
							}
						}
					}
				}
				else {
					if (player && oldHeld != undefined) switchSlot(oldHeld);
					delete tickLoop["Scaffold"];
				}
			}, "World", () => \`\${scaffoldtower[1] ? "Tower enabled" : "Horizontal Only"}\`);
			scaffoldtower = scaffold.addoption("Tower", Boolean, true);
			// scaffoldextend = scaffold.addoption("Extend", Number, 0);

			let timervalue;
			const timer = new Module("Timer", function(callback) {
				reloadTickLoop(callback ? 50 / timervalue[1] : 50);
			}, "World", () => \`\${timervalue[1]} MSPT\`);
			timervalue = timer.addoption("Value", Number, 1.2);
			new Module("Phase", function() {}, "World");

			const antiban = new Module("AntiBan", function() {}, "Misc", () => useAccountGen[1] ? "Gen" : "Non Account");
			useAccountGen = antiban.addoption("AccountGen", Boolean, false);
			accountGenEndpoint = antiban.addoption("GenServer", String, "http://localhost:8000/generate");
			antiban.toggle();
			new Module("AutoRejoin", function() {}, "Misc");
			new Module("AutoQueue", function() {}, "Minigames");
			new Module("AutoVote", function() {}, "Minigames");
			const chatdisabler = new Module("ChatDisabler", function() {}, "Misc", () => "Spam");
			chatdisablermsg = chatdisabler.addoption("Message", String, "youtube.com/c/7GrandDadVape");
			new Module("FilterBypass", function() {}, "Exploit", () => "\\\\");

			const survival = new Module("SurvivalMode", function(callback) {
				if (callback) {
					if (player) player.setGamemode(GameMode.fromId("survival"));
					survival.toggle();
				}
			}, "Misc", () => "Spoof");

			globalThis.${storeName}.modules = modules;
			globalThis.${storeName}.profile = "default";
		})();
	`);

	async function saveVapeConfig(profile) {
		if (!loadedConfig) return;
		let saveList = {};
		for (const [name, module] of Object.entries(unsafeWindow.globalThis[storeName].modules)) {
			saveList[name] = { enabled: module.enabled, bind: module.bind, options: {} };
			for (const [option, setting] of Object.entries(module.options)) {
				saveList[name].options[option] = setting[1];
			}
		}
		GM_setValue("vapeConfig" + (profile ?? unsafeWindow.globalThis[storeName].profile), JSON.stringify(saveList));
		GM_setValue("mainVapeConfig", JSON.stringify({ profile: unsafeWindow.globalThis[storeName].profile }));
	};

	async function loadVapeConfig(switched) {
		loadedConfig = false;
		const loadedMain = JSON.parse(await GM_getValue("mainVapeConfig", "{}")) ?? { profile: "default" };
		unsafeWindow.globalThis[storeName].profile = switched ?? loadedMain.profile;
		const loaded = JSON.parse(await GM_getValue("vapeConfig" + unsafeWindow.globalThis[storeName].profile, "{}"));
		if (!loaded) {
			loadedConfig = true;
			return;
		}

		for (const [name, module] of Object.entries(loaded)) {
			const realModule = unsafeWindow.globalThis[storeName].modules[name];
			if (!realModule) continue;
			if (realModule.enabled != module.enabled) realModule.toggle();
			if (realModule.bind != module.bind) realModule.setbind(module.bind);
			if (module.options) {
				for (const [option, setting] of Object.entries(module.options)) {
					const realOption = realModule.options[option];
					if (!realOption) continue;
					realOption[1] = setting;
				}
			}
		}
		loadedConfig = true;
	};

	async function exportVapeConfig() {
		navigator.clipboard.writeText(await GM_getValue("vapeConfig" + unsafeWindow.globalThis[storeName].profile, "{}"));
	};

	async function importVapeConfig() {
		const arg = await navigator.clipboard.readText();
		if (!arg) return;
		GM_setValue("vapeConfig" + unsafeWindow.globalThis[storeName].profile, arg);
		loadVapeConfig();
	};

	let loadedConfig = false;
	async function execute(src, oldScript) {
		Object.defineProperty(unsafeWindow.globalThis, storeName, { value: {}, enumerable: false });
		if (oldScript) oldScript.type = 'javascript/blocked';
		await fetch(src).then(e => e.text()).then(e => modifyCode(e));
		if (oldScript) oldScript.type = 'module';
		await new Promise((resolve) => {
			const loop = setInterval(async function () {
				if (unsafeWindow.globalThis[storeName].modules) {
					clearInterval(loop);
					resolve();
				}
			}, 10);
		});
		unsafeWindow.globalThis[storeName].saveVapeConfig = saveVapeConfig;
		unsafeWindow.globalThis[storeName].loadVapeConfig = loadVapeConfig;
		unsafeWindow.globalThis[storeName].exportVapeConfig = exportVapeConfig;
		unsafeWindow.globalThis[storeName].importVapeConfig = importVapeConfig;
		loadVapeConfig();
		setInterval(async function () {
			saveVapeConfig();
		}, 10000);
	}

	const publicUrl = "scripturl";
	// https://stackoverflow.com/questions/22141205/intercept-and-alter-a-sites-javascript-using-greasemonkey
	if (publicUrl == "scripturl") {
		if (navigator.userAgent.indexOf("Firefox") != -1) {
			window.addEventListener("beforescriptexecute", function (e) {
				if (e.target.src.includes("https://miniblox.io/assets/index")) {
					e.preventDefault();
					e.stopPropagation();
					execute(e.target.src);
				}
			}, false);
		}
		else {
			new MutationObserver(async (mutations, observer) => {
				let oldScript = mutations
					.flatMap(e => [...e.addedNodes])
					.filter(e => e.tagName == 'SCRIPT')
					.find(e => e.src.includes("https://miniblox.io/assets/index"));

				if (oldScript) {
					observer.disconnect();
					execute(oldScript.src, oldScript);
				}
			}).observe(document, {
				childList: true,
				subtree: true,
			});
		}
	}
	else {
		execute(publicUrl);
	}
})();
