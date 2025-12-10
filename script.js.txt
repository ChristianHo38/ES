// 游戏数据和配置
const gameData = {
    characters: {
        "6星": [
            "明星昴流", "冰鹰北斗", "日日树涉", "朔间零", "月永雷欧",
            "守泽千秋", "斋宫宗", "逆先夏目", "青叶纺"
        ],
        "5星": [
            "天祥院英智", "大神晃牙", "濑名泉", "鸣上岚", "深海奏汰",
            "仁兔成鸣", "莲巳敬人", "鬼龙红郎", "影片美伽", "三毛缟斑",
            "巴日和", "乱凪砂"
        ],
        "4星": [
            "游木真", "衣更真绪", "伏见弓弦", "姬宫桃李", "羽风薰",
            "乙狩阿多尼斯", "朔间凛月", "朱樱司", "南云铁虎", "仙石忍",
            "高峰翠", "真白友也", "紫之创", "天满光", "神崎飒马",
            "葵日向", "葵裕太", "春川宙", "涟纯", "七种茨"
        ],
        "3星": [
            "应援棒", "消毒剂", "应援服", "理智药"
        ]
    },

    sixStarLines: {
        "斋宫宗": "让你们见识这魅力吧，所谓接触到我的艺术就是如此。",
        "明星昴流": "作为一名偶像，我要成为闪闪发光的一等星⭐",
        "日日树涉": "为这个世界送上爱、与Amazing……⭐",
        "守泽千秋": "来吧，呼喊出来吧！不论何时英雄都会立刻赶赴你的身边噢！",
        "月永雷欧": "既是偶像又是作曲家！那就是我……🎶",
        "青叶纺": "作为一名偶像，我希望自己可以成为能给所有人带去梦想的存在。",
        "朔间零": "来吧，夜已深邃，吾辈的时间到来了……",
        "逆先夏目": "小猫咪，准备好被我施以魔法了吗a？",
        "冰鹰北斗": "我还能继续向更高的境界攀登，你就期待我的表现吧。"
    },

    baseRates: {
        "6星": 0.02,
        "5星": 0.12,
        "4星": 0.30,
        "3星": 0.56
    }
};

// 游戏状态
class GameState {
    constructor() {
        this.totalPulls = 0;
        this.sinceLast6Star = 0;
        this.sinceLastFeatured = 0;
        this.last6Star = null;
        this.lastFeatured = false;
        this.history = [];
        this.inventory = {
            "6星": {},
            "5星": {},
            "4星": {},
            "3星": {}
        };
        this.featured6Stars = this.selectFeaturedCharacters();
        this.currentTenPull = [];
        this.currentPullIndex = 0;
        this.isTenPull = false;
    }

    selectFeaturedCharacters() {
        const available6Stars = gameData.characters["6星"].filter(char => char !== "明星昴流");
        let featured = available6Stars.length >= 2 ? 
            [...new Set([...available6Stars.sort(() => 0.5 - Math.random()).slice(0, 2)])] : 
            [...available6Stars];
        
        if (featured.length < 2) {
            featured = [...gameData.characters["6星"]].sort(() => 0.5 - Math.random()).slice(0, 2);
        }
        
        featured.unshift("明星昴流");
        return featured.slice(0, 2);
    }

    getCurrent6StarRate() {
        if (this.sinceLast6Star >= 90) return 1.0;
        if (this.sinceLast6Star > 60) {
            const pityBonus = (this.sinceLast6Star - 60) * 0.02;
            return Math.min(1.0, gameData.baseRates["6星"] + pityBonus);
        }
        return gameData.baseRates["6星"];
    }

    check90PityGuarantee() {
        return this.sinceLast6Star >= 90;
    }

    checkFeaturedGuarantee() {
        return this.last6Star && !this.lastFeatured;
    }

    drawOne(guarantee5Star = false) {
        const current6StarRate = this.getCurrent6StarRate();
        let adjustedRates;

        if (guarantee5Star) {
            adjustedRates = {
                "6星": current6StarRate,
                "5星": 1.0 - current6StarRate,
                "4星": 0.0,
                "3星": 0.0
            };
        } else {
            const remainingProb = 1.0 - current6StarRate;
            if (remainingProb > 0) {
                const totalBase = gameData.baseRates["5星"] + gameData.baseRates["4星"] + gameData.baseRates["3星"];
                adjustedRates = {
                    "6星": current6StarRate,
                    "5星": (gameData.baseRates["5星"] / totalBase) * remainingProb,
                    "4星": (gameData.baseRates["4星"] / totalBase) * remainingProb,
                    "3星": (gameData.baseRates["3星"] / totalBase) * remainingProb
                };
            } else {
                adjustedRates = { "6星": 1.0, "5星": 0.0, "4星": 0.0, "3星": 0.0 };
            }
        }

        let selectedRarity = null;
        let rand = Math.random();
        let cumulative = 0.0;

        for (const [rarity, rate] of Object.entries(adjustedRates)) {
            cumulative += rate;
            if (rand <= cumulative + 1e-10) {
                selectedRarity = rarity;
                break;
            }
        }

        let character = null;
        let isFeatured = false;
        let triggeredPity = false;

        if (selectedRarity === "6星") {
            triggeredPity = this.check90PityGuarantee();
            const featuredGuaranteed = this.checkFeaturedGuarantee();

            if (featuredGuaranteed) {
                character = this.featured6Stars[Math.floor(Math.random() * this.featured6Stars.length)];
                isFeatured = true;
            } else {
                if (Math.random() < 0.6) {
                    character = this.featured6Stars[Math.floor(Math.random() * this.featured6Stars.length)];
                    isFeatured = true;
                } else {
                    const all6Stars = gameData.characters["6星"];
                    character = all6Stars[Math.floor(Math.random() * all6Stars.length)];
                    isFeatured = this.featured6Stars.includes(character);
                }
            }

            this.last6Star = character;
            this.lastFeatured = isFeatured;
            this.sinceLast6Star = 0;
            this.sinceLastFeatured = isFeatured ? 0 : this.sinceLastFeatured + 1;
        } else {
            const charactersList = gameData.characters[selectedRarity];
            character = charactersList[Math.floor(Math.random() * charactersList.length)];
            this.sinceLast6Star += 1;
            this.sinceLastFeatured += 1;
        }

        this.totalPulls += 1;

        return {
            count: this.totalPulls,
            rarity: selectedRarity,
            character: character,
            isFeatured: isFeatured,
            sinceLast6Star: this.sinceLast6Star,
            sinceLastFeatured: this.sinceLastFeatured,
            triggeredPity: triggeredPity,
            revealed: false
        };
    }

    revealResult(result) {
        result.revealed = true;
        this.history.push(result);

        const rarity = result.rarity;
        const character = result.character;
        if (!this.inventory[rarity][character]) {
            this.inventory[rarity][character] = 1;
        } else {
            this.inventory[rarity][character] += 1;
        }

        return result;
    }

    drawTen() {
        const results = [];
        let got5StarOrHigher = false;

        for (let i = 0; i < 10; i++) {
            if (i === 9 && !got5StarOrHigher) {
                results.push(this.drawOne(true));
            } else {
                const result = this.drawOne();
                if (result.rarity === "5星" || result.rarity === "6星") {
                    got5StarOrHigher = true;
                }
                results.push(result);
            }
        }

        return results;
    }
}

// 游戏核心
class GachaGame {
    constructor() {
        this.state = new GameState();
        this.currentScreen = 'main';
        this.isAnimationPlaying = false;
        this.dialogueTyping = null;
        this.initEventListeners();
        this.updateUI();
    }

    initEventListeners() {
        // 主界面按钮
        document.getElementById('single-pull-btn').addEventListener('click', () => this.startSinglePull());
        document.getElementById('ten-pull-btn').addEventListener('click', () => this.startTenPull());
        document.getElementById('stats-btn').addEventListener('click', () => this.showScreen('stats'));
        document.getElementById('history-btn').addEventListener('click', () => this.showScreen('history'));
        document.getElementById('pool-btn').addEventListener('click', () => this.showScreen('pool'));
        document.getElementById('reset-btn').addEventListener('click', () => this.resetGame());

        // 抽卡界面按钮
        document.getElementById('reveal-btn').addEventListener('click', () => this.revealCard());
        document.getElementById('skip-animation').addEventListener('click', () => this.skipAnimation());

        // 特效界面按钮
        document.getElementById('next-dialogue').addEventListener('click', () => this.showDialogue());

        // 台词界面按钮
        document.getElementById('next-result').addEventListener('click', () => this.showResult());

        // 结果界面按钮
        document.getElementById('result-continue').addEventListener('click', () => this.continueFromResult());

        // 十连总结界面按钮
        document.getElementById('summary-continue').addEventListener('click', () => this.showScreen('main'));

        // 返回按钮
        document.getElementById('back-to-main').addEventListener('click', () => this.showScreen('main'));
        document.getElementById('history-back').addEventListener('click', () => this.showScreen('main'));
        document.getElementById('pool-back').addEventListener('click', () => this.showScreen('main'));

        // 历史筛选
        document.getElementById('history-filter').addEventListener('change', () => this.updateHistory());
        document.getElementById('history-count').addEventListener('input', () => this.updateHistory());

        // 模态框
        document.getElementById('modal-confirm').addEventListener('click', () => this.modalConfirm());
        document.getElementById('modal-cancel').addEventListener('click', () => this.hideModal());
    }

    showScreen(screenName) {
        document.querySelectorAll('.screen').forEach(screen => {
            screen.classList.remove('active');
        });

        this.currentScreen = screenName;
        const screenElement = document.getElementById(`${screenName}-screen`);
        if (screenElement) {
            screenElement.classList.add('active');
        }

        if (screenName === 'main') {
            this.updateUI();
        } else if (screenName === 'stats') {
            this.updateStatsScreen();
        } else if (screenName === 'history') {
            this.updateHistory();
        } else if (screenName === 'pool') {
            this.updatePoolScreen();
        }
    }

    updateUI() {
        // 更新主界面统计
        document.getElementById('total-pulls').textContent = this.state.totalPulls;
        document.getElementById('since-6star').textContent = this.state.sinceLast6Star;
        document.getElementById('to-pity').textContent = 90 - this.state.sinceLast6Star;

        // 更新当期UP显示
        const upNames = this.state.featured6Stars.map(char => 
            `<span class="featured-name">${char}</span>`
        ).join(', ');
        document.getElementById('current-up-names').innerHTML = upNames;
    }

    startSinglePull() {
        this.state.isTenPull = false;
        this.state.currentTenPull = [this.state.drawOne()];
        this.state.currentPullIndex = 0;
        this.showGachaScreen(0, 1);
    }

    startTenPull() {
        this.state.isTenPull = true;
        this.state.currentTenPull = this.state.drawTen();
        this.state.currentPullIndex = 0;
        this.showGachaScreen(0, 10);
    }

    showGachaScreen(current, total) {
        const card = document.getElementById('current-card');
        card.classList.remove('revealed');
        
        document.getElementById('pull-count').textContent = `第 ${current + 1} 抽`;
        document.getElementById('pull-progress').textContent = total > 1 ? 
            `进度: ${current + 1}/${total}` : '单次抽取';
        
        this.showScreen('gacha');
        
        // 添加卡片动画
        setTimeout(() => {
            card.style.animation = 'none';
            setTimeout(() => {
                card.style.animation = 'pulse 2s infinite';
            }, 10);
        }, 100);
    }

    revealCard() {
        if (this.isAnimationPlaying) return;
        
        const currentResult = this.state.currentTenPull[this.state.currentPullIndex];
        this.isAnimationPlaying = true;
        
        // 卡片翻转动画
        const card = document.getElementById('current-card');
        card.style.transform = 'rotateY(180deg)';
        
        setTimeout(() => {
            card.classList.add('revealed');
            card.style.transform = 'rotateY(0deg)';
            
            if (currentResult.rarity === '6星') {
                setTimeout(() => {
                    this.showScreen('six-star-effect');
                    this.isAnimationPlaying = false;
                }, 1500);
            } else {
                setTimeout(() => {
                    this.state.revealResult(currentResult);
                    this.showResultScreen(currentResult);
                    this.isAnimationPlaying = false;
                }, 1000);
            }
        }, 500);
    }

    skipAnimation() {
        const currentResult = this.state.currentTenPull[this.state.currentPullIndex];
        
        if (currentResult.rarity === '6星') {
            this.showScreen('six-star-effect');
        } else {
            this.state.revealResult(currentResult);
            this.showResultScreen(currentResult);
        }
    }

    showDialogue() {
        const currentResult = this.state.currentTenPull[this.state.currentPullIndex];
        const character = currentResult.character;
        const dialogue = gameData.sixStarLines[character] || "欢迎加入偶像梦幻祭！";
        
        document.getElementById('dialogue-character').textContent = character;
        document.getElementById('dialogue-text').textContent = '';
        
        this.showScreen('dialogue');
        
        // 逐字打印台词
        let index = 0;
        this.dialogueTyping = setInterval(() => {
            const textElement = document.getElementById('dialogue-text');
            if (index < dialogue.length) {
                textElement.textContent += dialogue.charAt(index);
                index++;
            } else {
                clearInterval(this.dialogueTyping);
            }
        }, 50);
    }

    showResult() {
        if (this.dialogueTyping) {
            clearInterval(this.dialogueTyping);
            document.getElementById('dialogue-text').textContent = 
                gameData.sixStarLines[this.state.currentTenPull[this.state.currentPullIndex].character];
        }
        
        const currentResult = this.state.currentTenPull[this.state.currentPullIndex];
        this.state.revealResult(currentResult);
        this.showResultScreen(currentResult);
    }

    showResultScreen(result) {
        const rarityColors = {
            "6星": "rarity-6",
            "5星": "rarity-5", 
            "4星": "rarity-4",
            "3星": "rarity-3"
        };

        document.getElementById('result-pull-number').textContent = `第 ${result.count} 次抽取结果`;
        
        const rarityDisplay = document.getElementById('result-rarity');
        rarityDisplay.textContent = result.rarity;
        rarityDisplay.className = `rarity-display ${rarityColors[result.rarity]}`;
        
        document.getElementById('result-character-name').textContent = result.character;
        
        const featuredElement = document.getElementById('result-featured');
        if (result.isFeatured) {
            featuredElement.textContent = '☆ 当期UP角色 ☆';
            featuredElement.style.display = 'block';
        } else {
            featuredElement.style.display = 'none';
        }
        
        const pityElement = document.getElementById('result-pity');
        if (result.triggeredPity) {
            pityElement.textContent = '⚡ 触发90抽保底！';
            pityElement.style.display = 'block';
        } else {
            pityElement.style.display = 'none';
        }
        
        document.getElementById('result-next-6star').textContent = `${result.sinceLast6Star}抽`;
        document.getElementById('result-next-featured').textContent = `${result.sinceLastFeatured}抽`;
        
        this.showScreen('result');
    }

    continueFromResult() {
        this.state.currentPullIndex++;
        
        if (this.state.isTenPull && this.state.currentPullIndex < 10) {
            this.showGachaScreen(this.state.currentPullIndex, 10);
        } else if (this.state.isTenPull) {
            this.showTenSummary();
        } else {
            this.showScreen('main');
            this.updateUI();
        }
    }

    showTenSummary() {
        const results = this.state.currentTenPull;
        const rarityCounts = { "6星": 0, "5星": 0, "4星": 0, "3星": 0 };
        const sixStars = [];
        let triggeredPity = false;

        results.forEach(result => {
            rarityCounts[result.rarity]++;
            if (result.rarity === '6星') {
                sixStars.push(result);
                if (result.triggeredPity) triggeredPity = true;
            }
        });

        // 更新统计显示
        document.getElementById('summary-stats').innerHTML = `
            <div class="stat-item rarity-6">6星: ${rarityCounts["6星"]}个</div>
            <div class="stat-item rarity-5">5星: ${rarityCounts["5星"]}个</div>
            <div class="stat-item rarity-4">4星: ${rarityCounts["4星"]}个</div>
            <div class="stat-item rarity-3">3星: ${rarityCounts["3星"]}个</div>
        `;

        // 更新结果列表
        const resultsHtml = results.map(result => {
            let className = `result-item ${result.rarity === '6星' ? 'rarity-6' : 
                            result.rarity === '5星' ? 'rarity-5' :
                            result.rarity === '4星' ? 'rarity-4' : 'rarity-3'}`;
            
            if (result.isFeatured) className += ' featured';
            if (result.triggeredPity) className += ' pity';
            
            return `
                <div class="${className}">
                    <span>${result.character}</span>
                    <span>${result.rarity}</span>
                </div>
            `;
        }).join('');

        document.getElementById('summary-results').innerHTML = resultsHtml;

        // 更新计数器
        document.getElementById('summary-total-pulls').textContent = this.state.totalPulls;
        document.getElementById('summary-next-6star').textContent = `${this.state.sinceLast6Star}抽`;

        // 添加边框特效
        const container = document.querySelector('.summary-container');
        if (rarityCounts["6星"] > 0) {
            container.style.borderColor = '#8b5cf6';
            container.style.boxShadow = '0 0 30px rgba(139, 92, 246, 0.5)';
        } else if (rarityCounts["5星"] > 0) {
            container.style.borderColor = '#f59e0b';
            container.style.boxShadow = '0 0 30px rgba(245, 158, 11, 0.5)';
        } else {
            container.style.borderColor = 'rgba(255, 255, 255, 0.2)';
            container.style.boxShadow = 'none';
        }

        if (triggeredPity) {
            const header = document.querySelector('.summary-header');
            header.insertAdjacentHTML('beforeend', 
                '<div class="pity-notice">⚡ 本次十连触发了90抽保底！</div>');
        }

        this.showScreen('ten-summary');
    }

    updateStatsScreen() {
        document.getElementById('stats-total-pulls').textContent = this.state.totalPulls;
        document.getElementById('stats-since-6star').textContent = this.state.sinceLast6Star;
        document.getElementById('stats-to-pity').textContent = 90 - this.state.sinceLast6Star;
        
        // 计算6星总数
        let sixStarCount = 0;
        Object.values(this.state.inventory["6星"]).forEach(count => {
            sixStarCount += count;
        });
        document.getElementById('stats-6star-count').textContent = sixStarCount;

        // 更新库存显示
        this.updateInventoryDisplay();
    }

    updateInventoryDisplay() {
        const rarities = ["6星", "5星", "4星", "3星"];
        
        rarities.forEach(rarity => {
            const inventory = this.state.inventory[rarity];
            const count = Object.values(inventory).reduce((sum, val) => sum + val, 0);
            document.getElementById(`inventory-${rarity.replace('星', 'star')}-count`).textContent = `${count}个`;
            
            const itemsHtml = Object.entries(inventory).map(([character, count]) => {
                const isFeatured = this.state.featured6Stars.includes(character);
                const className = isFeatured ? 'inventory-item featured' : 'inventory-item';
                return `
                    <div class="${className}">
                        <span>${character}</span>
                        <span>×${count}</span>
                    </div>
                `;
            }).join('');
            
            document.getElementById(`inventory-${rarity.replace('星', 'star')}`).innerHTML = 
                itemsHtml || `<div class="inventory-item">暂无</div>`;
        });
    }

    updateHistory() {
        const filter = document.getElementById('history-filter').value;
        const count = parseInt(document.getElementById('history-count').value) || 20;
        
        let filteredHistory = [...this.state.history].reverse();
        
        if (filter === '6star') {
            filteredHistory = filteredHistory.filter(record => record.rarity === '6星');
        } else if (filter === '5star') {
            filteredHistory = filteredHistory.filter(record => record.rarity === '5星');
        } else if (filter === 'featured') {
            filteredHistory = filteredHistory.filter(record => record.isFeatured);
        }
        
        filteredHistory = filteredHistory.slice(0, count);
        
        const historyHtml = filteredHistory.map(record => {
            let className = `history-item ${record.rarity === '6星' ? 'rarity-6' : 
                            record.rarity === '5星' ? 'rarity-5' :
                            record.rarity === '4星' ? 'rarity-4' : 'rarity-3'}`;
            
            if (record.isFeatured) className += ' featured';
            if (record.triggeredPity) className += ' pity';
            
            return `
                <div class="${className}">
                    <span>第${record.count}抽</span>
                    <span>${record.character}</span>
                    <span>${record.rarity}${record.isFeatured ? ' [UP]' : ''}${record.triggeredPity ? ' [保底]' : ''}</span>
                </div>
            `;
        }).join('');
        
        document.getElementById('history-list').innerHTML = 
            historyHtml || '<div class="history-item">暂无记录</div>';
    }

    updatePoolScreen() {
        // 显示当期UP
        const featuredHtml = this.state.featured6Stars.map(character => 
            `<div class="featured-character">${character}</div>`
        ).join('');
        document.getElementById('pool-featured').innerHTML = featuredHtml;

        // 显示各星级角色
        const rarities = ["6星", "5星", "4星", "3星"];
        
        rarities.forEach(rarity => {
            const is6Star = rarity === "6星";
            const characters = gameData.characters[rarity];
            const containerId = `pool-${rarity.replace('星', 'star')}`;
            const isCharacter = rarity !== "3星";
            
            let itemsHtml = '';
            
            if (isCharacter) {
                itemsHtml = characters.map(character => {
                    const isFeatured = is6Star && this.state.featured6Stars.includes(character);
                    const className = isFeatured ? 'character-card featured' : 'character-card';
                    return `<div class="${className}">${character}</div>`;
                }).join('');
            } else {
                itemsHtml = characters.map(item => 
                    `<div class="item-card">${item}</div>`
                ).join('');
            }
            
            document.getElementById(containerId).innerHTML = itemsHtml;
        });
    }

    resetGame() {
        this.showModal('重新开始', '确定要重新开始游戏吗？所有数据将会丢失！', () => {
            this.state = new GameState();
            this.updateUI();
            this.showScreen('main');
            this.hideModal();
        });
    }

    showModal(title, message, onConfirm) {
        document.getElementById('modal-title').textContent = title;
        document.getElementById('modal-message').textContent = message;
        document.getElementById('modal').classList.add('active');
        
        const confirmBtn = document.getElementById('modal-confirm');
        const cancelBtn = document.getElementById('modal-cancel');
        
        // 移除旧的事件监听器
        const newConfirmBtn = confirmBtn.cloneNode(true);
        const newCancelBtn = cancelBtn.cloneNode(true);
        
        confirmBtn.parentNode.replaceChild(newConfirmBtn, confirmBtn);
        cancelBtn.parentNode.replaceChild(newCancelBtn, cancelBtn);
        
        // 添加新的事件监听器
        newConfirmBtn.addEventListener('click', () => {
            if (onConfirm) onConfirm();
            this.hideModal();
        });
        
        newCancelBtn.addEventListener('click', () => this.hideModal());
        
        // 更新引用
        document.getElementById('modal-confirm').onclick = () => {
            if (onConfirm) onConfirm();
            this.hideModal();
        };
        document.getElementById('modal-cancel').onclick = () => this.hideModal();
    }

    hideModal() {
        document.getElementById('modal').classList.remove('active');
    }

    modalConfirm() {
        this.hideModal();
    }
}

// 页面加载完成后初始化游戏
document.addEventListener('DOMContentLoaded', () => {
    window.game = new GachaGame();
});

// 保存游戏数据到本地存储
function saveGameData() {
    if (window.game && window.game.state) {
        const gameData = {
            state: window.game.state,
            timestamp: Date.now()
        };
        localStorage.setItem('gachaGameData', JSON.stringify(gameData));
    }
}

// 从本地存储加载游戏数据
function loadGameData() {
    const savedData = localStorage.getItem('gachaGameData');
    if (savedData) {
        try {
            const gameData = JSON.parse(savedData);
            // 这里可以添加数据验证和恢复逻辑
            return gameData.state;
        } catch (e) {
            console.error('加载游戏数据失败:', e);
        }
    }
    return null;
}

// 自动保存
setInterval(saveGameData, 30000); // 每30秒自动保存一次

// 页面卸载前保存
window.addEventListener('beforeunload', saveGameData);