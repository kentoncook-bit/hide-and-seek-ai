<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Hide and Seek AI – Player-Controlled Seeker</title>
<style>
body { margin:0; overflow:hidden; background:#111; color:white; font-family:sans-serif; }
#scoreboard {
    position:absolute; top:10px; left:10px; font-size:20px; background:rgba(0,0,0,0.5); padding:10px; border-radius:5px;
}
</style>
</head>
<body>
<div id="scoreboard">Caught: 0 | Remaining: 2 | History: []</div>
<script src="https://cdn.jsdelivr.net/npm/three@0.158.0/build/three.min.js"></script>
<script>
// === Scene & Renderer ===
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);

// --- Lights ---
scene.add(new THREE.AmbientLight(0xffffff,0.8));
const dirLight = new THREE.DirectionalLight(0xffffff,1.5);
dirLight.position.set(10,20,10);
scene.add(dirLight);

// --- Floor & Grid ---
const floor = new THREE.Mesh(new THREE.PlaneGeometry(50,50), new THREE.MeshStandardMaterial({color:0x222222}));
floor.rotation.x=-Math.PI/2;
scene.add(floor);
scene.add(new THREE.GridHelper(50,50,0x444444,0x444444));

// --- Walls ---
const walls=[];
function addWall(x,z,w,h,d){
    const wall=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),new THREE.MeshStandardMaterial({color:0x888888}));
    wall.position.set(x,h/2,z);
    scene.add(wall);
    walls.push(wall);
}
// Outer arena
addWall(0,-25,50,6,1);
addWall(0,25,50,6,1);
addWall(-25,0,1,6,50);
addWall(25,0,1,6,50);

// --- Hider area with two holes ---
const hiderArea = {x:15, z:-15, w:10, d:10};
const holeSize = 2;
const holePositions = [
    {x: hiderArea.x - hiderArea.w/4, z: hiderArea.z + hiderArea.d/2},
    {x: hiderArea.x + hiderArea.w/4, z: hiderArea.z + hiderArea.d/2}
];

function addHiderWallsTwoHoles(){
    const wallMat = new THREE.MeshStandardMaterial({color:0x888888});
    const leftWall = new THREE.Mesh(new THREE.BoxGeometry(1,6,hiderArea.d),wallMat);
    leftWall.position.set(hiderArea.x - hiderArea.w/2,3,hiderArea.z);
    scene.add(leftWall); walls.push(leftWall);
    const rightWall = new THREE.Mesh(new THREE.BoxGeometry(1,6,hiderArea.d),wallMat);
    rightWall.position.set(hiderArea.x + hiderArea.w/2,3,hiderArea.z);
    scene.add(rightWall); walls.push(rightWall);
    const backWall = new THREE.Mesh(new THREE.BoxGeometry(hiderArea.w,6,1),wallMat);
    backWall.position.set(hiderArea.x,3,hiderArea.z - hiderArea.d/2);
    scene.add(backWall); walls.push(backWall);
    // Front wall with holes
    const frontWall1 = new THREE.Mesh(new THREE.BoxGeometry(hiderArea.w/2 - holeSize/2,6,1),wallMat);
    frontWall1.position.set(hiderArea.x - hiderArea.w/4 - holeSize/4,3,hiderArea.z + hiderArea.d/2);
    scene.add(frontWall1); walls.push(frontWall1);
    const frontWall2 = new THREE.Mesh(new THREE.BoxGeometry(hiderArea.w/2 - holeSize/2,6,1),wallMat);
    frontWall2.position.set(hiderArea.x + hiderArea.w/4 + holeSize/4,3,hiderArea.z + hiderArea.d/2);
    scene.add(frontWall2); walls.push(frontWall2);
}
addHiderWallsTwoHoles();

// --- Boxes ---
const obstacles=[];
function addBox(x,z,w,h,d){
    const box=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),new THREE.MeshStandardMaterial({color:0x5555ff}));
    box.position.set(x,h/2,z);
    box.userData.velocity=new THREE.Vector3();
    scene.add(box);
    obstacles.push(box);
}
addBox(hiderArea.x-2,hiderArea.z,2,2,2);
addBox(hiderArea.x+2,hiderArea.z,2,2,2);

// --- Ramp for seekers ---
const ramp = new THREE.Mesh(new THREE.BoxGeometry(4,1,2),new THREE.MeshStandardMaterial({color:0x884400}));
ramp.position.set(hiderArea.x - hiderArea.w/2 - 2,0.5,hiderArea.z);
ramp.rotation.z = -Math.PI/8;
scene.add(ramp);

// --- Agent class with hole-blocking behavior ---
class Agent{
    constructor(color,isSeeker=false,spawnArea=null,constrainArea=null){
        this.body = new THREE.Mesh(new THREE.CylinderGeometry(0.5,0.5,1.5,12),new THREE.MeshStandardMaterial({color}));
        this.head = new THREE.Mesh(new THREE.SphereGeometry(0.35,12,12),new THREE.MeshStandardMaterial({color}));
        this.body.position.y = 0.75; this.head.position.y = 1.6;
        this.group = new THREE.Group(); this.group.add(this.body); this.group.add(this.head);

        this.spawnArea = spawnArea; this.constrainArea = constrainArea;
        if(spawnArea){
            this.group.position.set(
                spawnArea.x + (Math.random()-0.5)*spawnArea.w,0,
                spawnArea.z + (Math.random()-0.5)*spawnArea.d
            );
        } else {
            this.group.position.set((Math.random()-0.5)*20,0,(Math.random()-0.5)*20);
        }

        scene.add(this.group);
        this.isSeeker = isSeeker; this.velocity = new THREE.Vector3(); this.lastAction = 0; this.qTable = {};

        // Vision cone for seekers
        if(isSeeker){
            const coneGeo = new THREE.ConeGeometry(0.5,10,32,1,true);
            const coneMat = new THREE.MeshBasicMaterial({color:0xffff00,transparent:true,opacity:0.3,side:THREE.DoubleSide});
            this.vision = new THREE.Mesh(coneGeo,coneMat);
            this.vision.position.y = 1.6; this.vision.rotation.x=-Math.PI/2;
            scene.add(this.vision);
        }

        // Debug line & predicted move
        this.lineToHider = new THREE.Line(
            new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(), new THREE.Vector3()]),
            new THREE.LineBasicMaterial({color:0xff00ff})
        );
        scene.add(this.lineToHider);
        this.nextMoveArrow = new THREE.ArrowHelper(new THREE.Vector3(0,0,1), new THREE.Vector3(), 2, 0x00ff00);
        scene.add(this.nextMoveArrow);
    }

    getState(){ return `${Math.round(this.group.position.x)}|${Math.round(this.group.position.z)}|${this.isSeeker?'S':'H'}`; }

    chooseAction(){
        const state=this.getState();
        if(!this.qTable[state]) this.qTable[state]=[0,0,0,0];
        if(Math.random()<0.2) return Math.floor(Math.random()*4);
        const max=Math.max(...this.qTable[state]);
        return this.qTable[state].indexOf(max);
    }

    move(targets){
        this.lastAction = this.chooseAction();

        // --- Basic movement ---
        switch(this.lastAction){
            case 0: this.velocity.z -= 0.05; break;
            case 1: this.velocity.x += 0.05; break;
            case 2: this.velocity.z += 0.05; break;
            case 3: this.velocity.x -= 0.05; break;
        }

        // --- Hider hole-blocking behavior ---
        if(!this.isSeeker){
            const seekerNearby = targets.some(s=>this.group.position.distanceTo(s.group.position)<6);
            if(seekerNearby){
                let closestHole = holePositions.reduce((a,b)=>{
                    const da = this.group.position.clone().sub(a).length();
                    const db = this.group.position.clone().sub(b).length();
                    return da<db?a:b;
                });
                const dir = new THREE.Vector3(closestHole.x - this.group.position.x,0,closestHole.z - this.group.position.z);
                this.velocity.add(dir.multiplyScalar(0.05));

                let nearestBox = obstacles.reduce((a,b)=>{
                    const da = this.group.position.clone().sub(a.position).length();
                    const db = this.group.position.clone().sub(b.position).length();
                    return da<db?a:b;
                });
                const pushDir = new THREE.Vector3(closestHole.x - nearestBox.position.x,0,closestHole.z - nearestBox.position.z);
                nearestBox.userData.velocity.add(pushDir.multiplyScalar(0.03));
            }
        }

        this.group.position.add(this.velocity);
        this.velocity.multiplyScalar(0.9);

        if(this.constrainArea){
            this.group.position.x=Math.max(this.constrainArea.x - this.constrainArea.w/2,
                                           Math.min(this.constrainArea.x + this.constrainArea.w/2,this.group.position.x));
            this.group.position.z=Math.max(this.constrainArea.z - this.constrainArea.d/2,
                                           Math.min(this.constrainArea.z + this.constrainArea.d/2,this.group.position.z));
        }

        for(const box of obstacles){
            const dist=this.group.position.clone().sub(box.position).length();
            if(dist<1.5 && this.isSeeker) box.userData.velocity.add(this.velocity.clone().multiplyScalar(0.5));
        }

        if(this.isSeeker){
            this.vision.position.set(this.group.position.x,1.6,this.group.position.z);
            const dir = new THREE.Vector3();
            if(targets.length>0){
                const closest = targets.reduce((a,b)=>this.group.position.distanceTo(a.group.position)<this.group.position.distanceTo(b.group.position)?a:b);
                dir.subVectors(closest.group.position,this.group.position).normalize();
                this.lineToHider.geometry.setFromPoints([this.group.position.clone().setY(1.6), closest.group.position.clone().setY(1.6)]);
            }else dir.set(0,0,1);
            this.vision.rotation.y=Math.atan2(dir.x,dir.z);

            const nextDir = new THREE.Vector3();
            switch(this.lastAction){
                case 0: nextDir.set(0,0,-1); break;
                case 1: nextDir.set(1,0,0); break;
                case 2: nextDir.set(0,0,1); break;
                case 3: nextDir.set(-1,0,0); break;
            }
            this.nextMoveArrow.position.copy(this.group.position.clone().setY(0.75));
            this.nextMoveArrow.setDirection(nextDir.normalize());
        }
    }

    canSee(target){
        const dir = new THREE.Vector3().subVectors(target.group.position,this.group.position).normalize();
        const distance = this.group.position.distanceTo(target.group.position);
        const ray = new THREE.Raycaster(this.group.position.clone().setY(1.6), dir, 0, distance);
        for(const obs of obstacles.concat(walls)) if(ray.intersectObject(obs).length>0) return false;
        return distance<10;
    }

    updateQ(reward){
        const state=this.getState();
        if(!this.qTable[state]) this.qTable[state]=[0,0,0,0];
        const lr=0.2,discount=0.9;
        this.qTable[state][this.lastAction] = this.qTable[state][this.lastAction]*(1-lr)+lr*(reward+discount*Math.max(...this.qTable[state]));
    }

    resetPosition(){
        if(this.spawnArea){
            this.group.position.set(
                this.spawnArea.x + (Math.random()-0.5)*this.spawnArea.w,
                0,
                this.spawnArea.z + (Math.random()-0.5)*this.spawnArea.d
            );
        } else this.group.position.set((Math.random()-0.5)*20,0,(Math.random()-0.5)*20);
        this.velocity.set(0,0,0);
    }
}

// --- Spawn agents ---
const seekers=[new Agent(0xff0000,true),new Agent(0xff5555,true)];
let hiders=[new Agent(0x00ff00,false,hiderArea,hiderArea),new Agent(0x00aa00,false,hiderArea,hiderArea)];

// --- Player control ---
let playerControl={forward:0,right:0};
window.addEventListener('keydown',(e)=>{
    switch(e.key.toLowerCase()){
        case 'w': playerControl.forward=-0.2; break;
        case 's': playerControl.forward=0.2; break;
        case 'a': playerControl.right=-0.2; break;
        case 'd': playerControl.right=0.2; break;
    }
});
window.addEventListener('keyup',(e)=>{
    switch(e.key.toLowerCase()){
        case 'w': case 's': playerControl.forward=0; break;
        case 'a': case 'd': playerControl.right=0; break;
    }
});
function movePlayerSeeker(seeker){
    seeker.velocity.set(playerControl.right,0,playerControl.forward);
    seeker.group.position.add(seeker.velocity);
    seeker.group.position.x = Math.max(-24,Math.min(24,seeker.group.position.x));
    seeker.group.position.z = Math.max(-24,Math.min(24,seeker.group.position.z));
}

// --- Scoreboard ---
let caught=0;
let history=[];
const scoreboard=document.getElementById('scoreboard');
function updateScoreboard(){ 
    scoreboard.innerText=`Caught: ${caught} | Remaining: ${hiders.length} | History: [${history.join(', ')}]`; 
}

// --- Rewards ---
function getReward(agent){
    if(agent.isSeeker){
        let reward=-0.01;
        for(const h of hiders.slice()){
            if(agent.canSee(h)){ reward+=0.05; }
            if(agent.group.position.distanceTo(h.group.position)<1.2){
                reward+=10;
                hiders.splice(hiders.indexOf(h),1);
                caught++; updateScoreboard();
            }
        }
        return reward;
    } else {
        let reward = 0;
        for(const s of seekers){ reward += agent.group.position.distanceTo(s.group.position)*0.01; }
        for(const hole of holePositions){
            for(const box of obstacles){
                if(box.position.clone().sub(hole).length()<1) reward+=0.5;
            }
        }
        return reward;
    }
}

// --- Reset game ---
function resetGame(){
    history.push(caught);
    hiders=[new Agent(0x00ff00,false,hiderArea,hiderArea),new Agent(0x00aa00,false,hiderArea,hiderArea)];
    seekers.forEach(s=>s.resetPosition());
    caught=0; updateScoreboard();
}

// --- Camera ---
let cameraMode = "top"; 
window.addEventListener('keydown', (e)=>{ if(e.key.toLowerCase()==='c') cameraMode = cameraMode==='top'?'follow':'top'; });
function updateCamera(mainSeeker){
    if(cameraMode==="top"){
        camera.position.lerp(new THREE.Vector3(0,50,0),0.1);
        camera.lookAt(0,0,0);
        camera.up.set(0,0,-1);
    } else {
        const targetPos = mainSeeker.group.position.clone();
        const camPos = new THREE.Vector3(targetPos.x, targetPos.y+15, targetPos.z+20);
        camera.position.lerp(camPos,0.1);
        camera.lookAt(targetPos.clone().setY(0));
        camera.up.set(0,1,0);
    }
}

// --- Animate ---
function animate(){
    requestAnimationFrame(animate);
    const mainSeeker = seekers[0];

    // Player-controlled seeker
    movePlayerSeeker(mainSeeker);

    // AI seeker
    seekers[1].move(hiders);

    hiders.forEach(h=>h.move(seekers));

    [...seekers,...hiders].forEach(a=> a.updateQ(getReward(a)));

    obstacles.forEach(box=>{
        box.position.add(box.userData.velocity);
        box.userData.velocity.multiplyScalar(0.95);
        box.position.x=Math.max(hiderArea.x - hiderArea.w/2, Math.min(hiderArea.x + hiderArea.w/2, box.position.x));
        box.position.z=Math.max(hiderArea.z - hiderArea.d/2, Math.min(hiderArea.z + hiderArea.d/2, box.position.z));
    });

    updateCamera(mainSeeker);
    if(hiders.length===0) resetGame();
    renderer.render(scene,camera);
}
animate();
</script>
</body>
</html>
