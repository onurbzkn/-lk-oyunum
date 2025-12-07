# -lk-oyunum
1|from fastapi import FastAPI, APIRouter, HTTPException, Depends
2|from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
3|from dotenv import load_dotenv
4|from starlette.middleware.cors import CORSMiddleware
5|from motor.motor_asyncio import AsyncIOMotorClient
6|import os
7|import logging
8|from pathlib import Path
9|from pydantic import BaseModel, Field
10|from typing import List, Optional, Dict, Any
11|import uuid
12|from datetime import datetime, timedelta
13|import bcrypt
14|import jwt
15|import random
16|
17|ROOT_DIR = Path(__file__).parent
18|load_dotenv(ROOT_DIR / '.env')
19|
20|# MongoDB connection
21|mongo_url = os.environ['MONGO_URL']
22|client = AsyncIOMotorClient(mongo_url)
23|db = client[os.environ['DB_NAME']]
24|
25|# JWT Secret
26|JWT_SECRET = os.environ.get('JWT_SECRET', 'batak-secret-key-2025')
27|JWT_ALGORITHM = 'HS256'
28|
29|# Create the main app
30|app = FastAPI()
31|api_router = APIRouter(prefix="/api")
32|security = HTTPBearer()
33|
34|# ==================== MODELS ====================
35|
36|class RegisterRequest(BaseModel):
37|    username: str
38|    password: str
39|
40|class LoginRequest(BaseModel):
41|    username: str
42|    password: str
43|
44|class LoginResponse(BaseModel):
45|    token: str
46|    user: Dict[str, Any]
47|
48|class UserProfile(BaseModel):
49|    username: str
50|    points: int
51|    unlocked_cards: List[int]
52|    unlocked_backgrounds: List[int]
53|    current_card: int
54|    current_background: int
55|    total_games: int
56|    total_wins: int
57|
58|class UnlockRequest(BaseModel):
59|    item_type: str  # 'card' or 'background'
60|    item_id: int
61|
62|class SelectRequest(BaseModel):
63|    item_type: str
64|    item_id: int
65|
66|class StartGameRequest(BaseModel):
67|    difficulty: str  # 'easy', 'medium', 'hard'
68|
69|class BidRequest(BaseModel):
70|    bid_amount: int
71|    trump_suit: Optional[str] = None  # 'hearts', 'diamonds', 'clubs', 'spades'
72|
73|class PlayCardRequest(BaseModel):
74|    card: str  # e.g., "AS" (Ace of Spades)
75|
76|# ==================== SHOP CONFIG ====================
77|
78|CARD_DESIGNS = [
79|    {"id": 0, "name": "Klasik", "price": 0, "unlocked_by_default": True},
80|    {"id": 1, "name": "Modern", "price": 300},
81|    {"id": 2, "name": "Neon", "price": 400},
82|    {"id": 3, "name": "Altın", "price": 500},
83|    {"id": 4, "name": "Gümüş", "price": 600},
84|    {"id": 5, "name": "Kristal", "price": 700},
85|    {"id": 6, "name": "Ateş", "price": 800},
86|    {"id": 7, "name": "Buz", "price": 900},
87|    {"id": 8, "name": "Galaksi", "price": 1000},
88|    {"id": 9, "name": "Elmas", "price": 1200}
89|]
90|
91|BACKGROUNDS = [
92|    {"id": 0, "name": "Yeşil Masa", "price": 0, "unlocked_by_default": True},
93|    {"id": 1, "name": "Mavi Masa", "price": 300},
94|    {"id": 2, "name": "Kırmızı Masa", "price": 400},
95|    {"id": 3, "name": "Ahşap Masa", "price": 500},
96|    {"id": 4, "name": "Mermer Masa", "price": 600}
97|]
98|
99|POINTS_BY_DIFFICULTY = {
100|    'easy': 30,
101|    'medium': 50,
102|    'hard': 100
103|}
104|
105|# ==================== HELPER FUNCTIONS ====================
106|
107|def hash_password(password: str) -> str:
108|    return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
109|
110|def verify_password(password: str, hashed: str) -> bool:
111|    return bcrypt.checkpw(password.encode('utf-8'), hashed.encode('utf-8'))
112|
113|def create_token(username: str) -> str:
114|    payload = {
115|        'username': username,
116|        'exp': datetime.utcnow() + timedelta(days=30)
117|    }
118|    return jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGORITHM)
119|
120|async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
121|    try:
122|        token = credentials.credentials
123|        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
124|        username = payload.get('username')
125|        if not username:
126|            raise HTTPException(status_code=401, detail="Invalid token")
127|        
128|        user = await db.users.find_one({'username': username})
129|        if not user:
130|            raise HTTPException(status_code=401, detail="User not found")
131|        
132|        return user
133|    except jwt.ExpiredSignatureError:
134|        raise HTTPException(status_code=401, detail="Token expired")
135|    except jwt.InvalidTokenError:
136|        raise HTTPException(status_code=401, detail="Invalid token")
137|
138|# ==================== BATAK GAME LOGIC ====================
139|
140|class BatakGame:
141|    def __init__(self, difficulty: str):
142|        self.difficulty = difficulty
143|        self.deck = self.create_deck()
144|        self.players = ['user', 'ai1', 'ai2', 'ai3']
145|        self.hands = {player: [] for player in self.players}
146|        self.current_round = []
147|        self.scores = {player: 0 for player in self.players}
148|        self.tricks_won = {player: 0 for player in self.players}
149|        self.current_player_index = 0
150|        self.dealer_index = 0
151|        self.bidding_phase = True
152|        self.bids = {}
153|        self.trump_suit = None
154|        self.lead_suit = None
155|        self.round_number = 0
156|        self.game_over = False
157|        self.winner = None
158|        
159|    def create_deck(self):
160|        suits = ['hearts', 'diamonds', 'clubs', 'spades']
161|        ranks = ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A']
162|        deck = [f"{rank}{suit[0].upper()}" for suit in suits for rank in ranks]
163|        random.shuffle(deck)
164|        return deck
165|    
166|    def deal_cards(self):
167|        for i, card in enumerate(self.deck):
168|            player = self.players[i % 4]
169|            self.hands[player].append(card)
170|        
171|        # Sort hands
172|        for player in self.players:
173|            self.hands[player] = self.sort_hand(self.hands[player])
174|    
175|    def sort_hand(self, hand):
176|        suit_order = {'H': 0, 'D': 1, 'C': 2, 'S': 3}
177|        rank_order = {'2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9, '10': 10, 'J': 11, 'Q': 12, 'K': 13, 'A': 14}
178|        
179|        return sorted(hand, key=lambda card: (suit_order.get(card[-1], 0), rank_order.get(card[:-1], 0)))
180|    
181|    def get_card_value(self, card):
182|        rank = card[:-1]
183|        rank_values = {'2': 2, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9, '10': 10, 'J': 11, 'Q': 12, 'K': 13, 'A': 14}
184|        return rank_values.get(rank, 0)
185|    
186|    def get_card_suit(self, card):
187|        return card[-1]
188|    
189|    def ai_bid(self, player):
190|        hand = self.hands[player]
191|        
192|        if self.difficulty == 'easy':
193|            return random.randint(1, 7)
194|        elif self.difficulty == 'medium':
195|            high_cards = sum(1 for card in hand if self.get_card_value(card) >= 11)
196|            return min(max(high_cards, 1), 13)
197|        else:  # hard
198|            high_cards = sum(1 for card in hand if self.get_card_value(card) >= 12)
199|            aces = sum(1 for card in hand if card[:-1] == 'A')
200|            return min(max(high_cards + aces, 2), 13)
201|    
202|    def ai_choose_trump(self, player):
203|        hand = self.hands[player]
204|        suit_counts = {'H': 0, 'D': 0, 'C': 0, 'S': 0}
205|        
206|        for card in hand:
207|            suit = self.get_card_suit(card)
208|            suit_counts[suit] += 1
209|        
210|        max_suit = max(suit_counts, key=suit_counts.get)
211|        suit_map = {'H': 'hearts', 'D': 'diamonds', 'C': 'clubs', 'S': 'spades'}
212|        return suit_map[max_suit]
213|    
214|    def ai_play_card(self, player):
215|        hand = self.hands[player]
216|        valid_cards = self.get_valid_cards(player)
217|        
218|        if not valid_cards:
219|            return None
220|        
221|        if self.difficulty == 'easy':
222|            return random.choice(valid_cards)
223|        elif self.difficulty == 'medium':
224|            # Try to win the trick if possible
225|            if self.current_round:
226|                winning_card = self.get_winning_card()
227|                higher_cards = [c for c in valid_cards if self.is_card_higher(c, winning_card)]
228|                if higher_cards:
229|                    return min(higher_cards, key=lambda c: self.get_card_value(c))
230|            return random.choice(valid_cards)
231|        else:  # hard
232|            # Advanced strategy
233|            if not self.current_round:
234|                # Lead with high cards
235|                return max(valid_cards, key=lambda c: self.get_card_value(c))
236|            else:
237|                winning_card = self.get_winning_card()
238|                higher_cards = [c for c in valid_cards if self.is_card_higher(c, winning_card)]
239|                if higher_cards:
240|                    return min(higher_cards, key=lambda c: self.get_card_value(c))
241|                else:
242|                    return min(valid_cards, key=lambda c: self.get_card_value(c))
243|    
244|    def get_valid_cards(self, player):
245|        hand = self.hands[player]
246|        
247|        if not self.current_round:
248|            return hand
249|        
250|        lead_suit = self.lead_suit
251|        cards_in_suit = [card for card in hand if self.get_card_suit(card) == lead_suit[0]]
252|        
253|        if not cards_in_suit:
254|            # No cards in lead suit, can play anything
255|            return hand
256|        
257|        # BATAK KURALI: Aynı renkten kartın varsa, daha büyük kart varsa onu atmak zorundasın
258|        winning_card = self.get_winning_card()
259|        
260|        # Lead suit kartlarından daha büyük olanları bul
261|        higher_cards = [card for card in cards_in_suit if self.is_card_higher(card, winning_card)]
262|        
263|        if higher_cards:
264|            # Daha büyük kartın varsa, sadece onları oynayabilirsin
265|            return higher_cards
266|        
267|        # Daha büyük kartın yoksa, aynı renkteki herhangi bir kartı oynayabilirsin
268|        return cards_in_suit
269|    
270|    def is_card_higher(self, card1, card2):
271|        suit1 = self.get_card_suit(card1)
272|        suit2 = self.get_card_suit(card2)
273|        
274|        trump_initial = self.trump_suit[0].upper() if self.trump_suit else None
275|        
276|        if suit1 == trump_initial and suit2 != trump_initial:
277|            return True
278|        if suit2 == trump_initial and suit1 != trump_initial:
279|            return False
280|        if suit1 != suit2:
281|            return False
282|        
283|        return self.get_card_value(card1) > self.get_card_value(card2)
284|    
285|    def get_winning_card(self):
286|        if not self.current_round:
287|            return None
288|        
289|        winning_card = self.current_round[0]['card']
290|        for play in self.current_round[1:]:
291|            if self.is_card_higher(play['card'], winning_card):
292|                winning_card = play['card']
293|        
294|        return winning_card
295|    
296|    def get_trick_winner(self):
297|        winning_card = self.get_winning_card()
298|        for play in self.current_round:
299|            if play['card'] == winning_card:
300|                return play['player']
301|        return None
302|    
303|    def to_dict(self):
304|        return {
305|            'difficulty': self.difficulty,
306|            'hands': self.hands,
307|            'current_round': self.current_round,
308|            'scores': self.scores,
309|            'tricks_won': self.tricks_won,
310|            'current_player': self.players[self.current_player_index],
311|            'bidding_phase': self.bidding_phase,
312|            'bids': self.bids,
313|            'trump_suit': self.trump_suit,
314|            'lead_suit': self.lead_suit,
315|            'round_number': self.round_number,
316|            'game_over': self.game_over,
317|            'winner': self.winner
318|        }
319|
320|# Store active games in memory (in production, use Redis or similar)
321|active_games = {}
322|
323|# ==================== AUTH ROUTES ====================
324|
325|@api_router.post("/auth/register")
326|async def register(request: RegisterRequest):
327|    # Check if user exists
328|    existing_user = await db.users.find_one({'username': request.username})
329|    if existing_user:
330|        raise HTTPException(status_code=400, detail="Username already exists")
331|    
332|    # Create new user
333|    user_data = {
334|        'username': request.username,
335|        'password_hash': hash_password(request.password),
336|        'points': 0,
337|        'unlocked_cards': [0],  # Default card design
338|        'unlocked_backgrounds': [0],  # Default background
339|        'current_card': 0,
340|        'current_background': 0,
341|        'total_games': 0,
342|        'total_wins': 0,
343|        'created_at': datetime.utcnow()
344|    }
345|    
346|    await db.users.insert_one(user_data)
347|    
348|    token = create_token(request.username)
349|    user_data.pop('password_hash')
350|    user_data.pop('_id', None)
351|    
352|    return LoginResponse(token=token, user=user_data)
353|
354|@api_router.post("/auth/login")
355|async def login(request: LoginRequest):
356|    # Find user
357|    user = await db.users.find_one({'username': request.username})
358|    if not user:
359|        raise HTTPException(status_code=401, detail="Invalid credentials")
360|    
361|    # Verify password
362|    if not verify_password(request.password, user['password_hash']):
363|        raise HTTPException(status_code=401, detail="Invalid credentials")
364|    
365|    token = create_token(request.username)
366|    user.pop('password_hash')
367|    user.pop('_id', None)
368|    
369|    return LoginResponse(token=token, user=user)
370|
371|# ==================== USER ROUTES ====================
372|
373|@api_router.get("/user/profile")
374|async def get_profile(current_user: dict = Depends(get_current_user)):
375|    current_user.pop('password_hash', None)
376|    current_user.pop('_id', None)
377|    return current_user
378|
379|@api_router.get("/user/stats")
380|async def get_stats(current_user: dict = Depends(get_current_user)):
381|    total_games = current_user.get('total_games', 0)
382|    total_wins = current_user.get('total_wins', 0)
383|    win_rate = (total_wins / total_games * 100) if total_games > 0 else 0
384|    
385|    return {
386|        'total_games': total_games,
387|        'total_wins': total_wins,
388|        'win_rate': round(win_rate, 2),
389|        'points': current_user.get('points', 0)
390|    }
391|
392|# ==================== SHOP ROUTES ====================
393|
394|@api_router.get("/shop/items")
395|async def get_shop_items(current_user: dict = Depends(get_current_user)):
396|    unlocked_cards = current_user.get('unlocked_cards', [0])
397|    unlocked_backgrounds = current_user.get('unlocked_backgrounds', [0])
398|    
399|    cards = [
400|        {**design, 'unlocked': design['id'] in unlocked_cards}
401|        for design in CARD_DESIGNS
402|    ]
403|    
404|    backgrounds = [
405|        {**bg, 'unlocked': bg['id'] in unlocked_backgrounds}
406|        for bg in BACKGROUNDS
407|    ]
408|    
409|    return {
410|        'cards': cards,
411|        'backgrounds': backgrounds,
412|        'user_points': current_user.get('points', 0)
413|    }
414|
415|@api_router.post("/shop/unlock")
416|async def unlock_item(request: UnlockRequest, current_user: dict = Depends(get_current_user)):
417|    username = current_user['username']
418|    
419|    if request.item_type == 'card':
420|        items = CARD_DESIGNS
421|        unlocked_field = 'unlocked_cards'
422|    elif request.item_type == 'background':
423|        items = BACKGROUNDS
424|        unlocked_field = 'unlocked_backgrounds'
425|    else:
426|        raise HTTPException(status_code=400, detail="Invalid item type")
427|    
428|    # Find item
429|    item = next((i for i in items if i['id'] == request.item_id), None)
430|    if not item:
431|        raise HTTPException(status_code=404, detail="Item not found")
432|    
433|    # Check if already unlocked
434|    if request.item_id in current_user.get(unlocked_field, []):
435|        raise HTTPException(status_code=400, detail="Item already unlocked")
436|    
437|    # Check if user has enough points
438|    if current_user.get('points', 0) < item['price']:
439|        raise HTTPException(status_code=400, detail="Not enough points")
440|    
441|    # Unlock item
442|    new_points = current_user['points'] - item['price']
443|    await db.users.update_one(
444|        {'username': username},
445|        {
446|            '$push': {unlocked_field: request.item_id},
447|            '$set': {'points': new_points}
448|        }
449|    )
450|    
451|    return {'success': True, 'new_points': new_points}
452|
453|@api_router.post("/shop/select")
454|async def select_item(request: SelectRequest, current_user: dict = Depends(get_current_user)):
455|    username = current_user['username']
456|    
457|    if request.item_type == 'card':
458|        unlocked_field = 'unlocked_cards'
459|        current_field = 'current_card'
460|    elif request.item_type == 'background':
461|        unlocked_field = 'unlocked_backgrounds'
462|        current_field = 'current_background'
463|    else:
464|        raise HTTPException(status_code=400, detail="Invalid item type")
465|    
466|    # Check if item is unlocked
467|    if request.item_id not in current_user.get(unlocked_field, []):
468|        raise HTTPException(status_code=400, detail="Item not unlocked")
469|    
470|    # Select item
471|    await db.users.update_one(
472|        {'username': username},
473|        {'$set': {current_field: request.item_id}}
474|    )
475|    
476|    return {'success': True}
477|
478|# ==================== GAME ROUTES ====================
479|
480|@api_router.post("/game/start")
481|async def start_game(request: StartGameRequest, current_user: dict = Depends(get_current_user)):
482|    username = current_user['username']
483|    
484|    if request.difficulty not in ['easy', 'medium', 'hard']:
485|        raise HTTPException(status_code=400, detail="Invalid difficulty")
486|    
487|    # Create new game
488|    game = BatakGame(request.difficulty)
489|    game.deal_cards()
490|    
491|    # Store game
492|    game_id = str(uuid.uuid4())
493|    active_games[f"{username}_{game_id}"] = game
494|    
495|    return {
496|        'game_id': game_id,
497|        'state': game.to_dict()
498|    }
499|
500|@api_router.post("/game/{game_id}/bid")
501|async def place_bid(game_id: str, request: BidRequest, current_user: dict = Depends(get_current_user)):
502|    username = current_user['username']
503|    game_key = f"{username}_{game_id}"
504|    
505|    if game_key not in active_games:
506|        raise HTTPException(status_code=404, detail="Game not found")
507|    
508|    game = active_games[game_key]
509|    
510|    if not game.bidding_phase:
511|        raise HTTPException(status_code=400, detail="Not in bidding phase")
512|    
513|    if request.bid_amount < 1 or request.bid_amount > 13:
514|        raise HTTPException(status_code=400, detail="Invalid bid amount")
515|    
516|    # User places bid
517|    game.bids['user'] = request.bid_amount
518|    
519|    # AI players place bids
520|    for ai_player in ['ai1', 'ai2', 'ai3']:
521|        game.bids[ai_player] = game.ai_bid(ai_player)
522|    
523|    # Find highest bidder
524|    highest_bidder = max(game.bids, key=game.bids.get)
525|    
526|    # Set trump suit
527|    if highest_bidder == 'user':
528|        if not request.trump_suit:
529|            raise HTTPException(status_code=400, detail="Trump suit required")
530|        game.trump_suit = request.trump_suit
531|    else:
532|        game.trump_suit = game.ai_choose_trump(highest_bidder)
533|    
534|    game.bidding_phase = False
535|    game.current_player_index = game.players.index(highest_bidder)
536|    
537|    return {
538|        'success': True,
539|        'bids': game.bids,
540|        'highest_bidder': highest_bidder,
541|        'trump_suit': game.trump_suit,
542|        'state': game.to_dict()
543|    }
544|
545|@api_router.post("/game/{game_id}/play")
546|async def play_card(game_id: str, request: PlayCardRequest, current_user: dict = Depends(get_current_user)):
547|    username = current_user['username']
548|    game_key = f"{username}_{game_id}"
549|    
550|    if game_key not in active_games:
551|        raise HTTPException(status_code=404, detail="Game not found")
552|    
553|    game = active_games[game_key]
554|    
555|    if game.bidding_phase:
556|        raise HTTPException(status_code=400, detail="Still in bidding phase")
557|    
558|    if game.game_over:
559|        raise HTTPException(status_code=400, detail="Game is over")
560|    
561|    current_player = game.players[game.current_player_index]
562|    
563|    if current_player != 'user':
564|        raise HTTPException(status_code=400, detail="Not your turn")
565|    
566|    # Validate card
567|    if request.card not in game.hands['user']:
568|        raise HTTPException(status_code=400, detail="Invalid card")
569|    
570|    valid_cards = game.get_valid_cards('user')
571|    if request.card not in valid_cards:
572|        raise HTTPException(status_code=400, detail="Cannot play this card")
573|    
574|    # Play card
575|    if not game.current_round:
576|        game.lead_suit = game.get_card_suit(request.card)
577|    
578|    game.current_round.append({'player': 'user', 'card': request.card})
579|    game.hands['user'].remove(request.card)
580|    game.current_player_index = (game.current_player_index + 1) % 4
581|    
582|    # AI players play
583|    ai_moves = []
584|    while len(game.current_round) < 4:
585|        current_player = game.players[game.current_player_index]
586|        ai_card = game.ai_play_card(current_player)
587|        
588|        if not game.current_round:
589|            game.lead_suit = game.get_card_suit(ai_card)
590|        
591|        game.current_round.append({'player': current_player, 'card': ai_card})
592|        game.hands[current_player].remove(ai_card)
593|        ai_moves.append({'player': current_player, 'card': ai_card})
594|        game.current_player_index = (game.current_player_index + 1) % 4
595|    
596|    # Determine trick winner
597|    trick_winner = game.get_trick_winner()
598|    game.tricks_won[trick_winner] += 1
599|    
600|    # Clear round
601|    game.current_round = []
602|    game.lead_suit = None
603|    game.current_player_index = game.players.index(trick_winner)
604|    game.round_number += 1
605|    
606|    # Check if game is over
607|    if game.round_number >= 13:
608|        game.game_over = True
609|        
610|        # Determine winner (most tricks)
611|        game.winner = max(game.tricks_won, key=game.tricks_won.get)
612|        
613|        # Award points if user won
614|        if game.winner == 'user':
615|            points_earned = POINTS_BY_DIFFICULTY[game.difficulty]
616|            await db.users.update_one(
617|                {'username': username},
618|                {
619|                    '$inc': {'points': points_earned, 'total_games': 1, 'total_wins': 1}
620|                }
621|            )
622|        else:
623|            await db.users.update_one(
624|                {'username': username},
625|                {'$inc': {'total_games': 1}}
626|            )
627|        
628|        # Save game history
629|        await db.game_history.insert_one({
630|            'username': username,
631|            'difficulty': game.difficulty,
632|            'result': 'win' if game.winner == 'user' else 'loss',
633|            'points_earned': POINTS_BY_DIFFICULTY[game.difficulty] if game.winner == 'user' else 0,
634|            'tricks_won': game.tricks_won['user'],
635|            'timestamp': datetime.utcnow()
636|        })
637|        
638|        # Clean up game
639|        del active_games[game_key]
640|    
641|    # Check if round is completed
642|    round_completed = len(game.current_round) == 0
643|    
644|    return {
645|        'success': True,
646|        'ai_moves': ai_moves,
647|        'trick_winner': trick_winner,
648|        'round_completed': round_completed,
649|        'state': game.to_dict()
650|    }
651|
652|@api_router.post("/game/{game_id}/play-ai")
653|async def play_ai_turn(game_id: str, current_user: dict = Depends(get_current_user)):
654|    username = current_user['username']
655|    game_key = f"{username}_{game_id}"
656|    
657|    if game_key not in active_games:
658|        raise HTTPException(status_code=404, detail="Game not found")
659|    
660|    game = active_games[game_key]
661|    
662|    if game.bidding_phase:
663|        raise HTTPException(status_code=400, detail="Still in bidding phase")
664|    
665|    if game.game_over:
666|        raise HTTPException(status_code=400, detail="Game is over")
667|    
668|    current_player = game.players[game.current_player_index]
669|    
670|    if current_player == 'user':
671|        raise HTTPException(status_code=400, detail="It's user's turn")
672|    
673|    # AI plays a card
674|    ai_card = game.ai_play_card(current_player)
675|    
676|    if not game.current_round:
677|        game.lead_suit = game.get_card_suit(ai_card)
678|    
679|    game.current_round.append({'player': current_player, 'card': ai_card})
680|    game.hands[current_player].remove(ai_card)
681|    game.current_player_index = (game.current_player_index + 1) % 4
682|    
683|    trick_winner = None
684|    round_completed = False
685|    
686|    # Check if round is complete
687|    if len(game.current_round) == 4:
688|        trick_winner = game.get_trick_winner()
689|        game.tricks_won[trick_winner] += 1
690|        
691|        # Clear round
692|        game.current_round = []
693|        game.lead_suit = None
694|        game.current_player_index = game.players.index(trick_winner)
695|        game.round_number += 1
696|        round_completed = True
697|        
698|        # Check if game is over
699|        if game.round_number >= 13:
700|            game.game_over = True
701|            game.winner = max(game.tricks_won, key=game.tricks_won.get)
702|            
703|            # Award points
704|            if game.winner == 'user':
705|                points_earned = POINTS_BY_DIFFICULTY[game.difficulty]
706|                await db.users.update_one(
707|                    {'username': username},
708|                    {'$inc': {'points': points_earned, 'total_games': 1, 'total_wins': 1}}
709|                )
710|            else:
711|                await db.users.update_one(
712|                    {'username': username},
713|                    {'$inc': {'total_games': 1}}
714|                )
715|            
716|            # Save game history
717|            await db.game_history.insert_one({
718|                'username': username,
719|                'difficulty': game.difficulty,
720|                'result': 'win' if game.winner == 'user' else 'loss',
721|                'points_earned': POINTS_BY_DIFFICULTY[game.difficulty] if game.winner == 'user' else 0,
722|                'tricks_won': game.tricks_won['user'],
723|                'timestamp': datetime.utcnow()
724|            })
725|            
726|            # Clean up game
727|            del active_games[game_key]
728|    
729|    return {
730|        'success': True,
731|        'ai_move': {'player': current_player, 'card': ai_card},
732|        'trick_winner': trick_winner,
733|        'round_completed': round_completed,
734|        'state': game.to_dict()
735|    }
736|
737|@api_router.get("/game/{game_id}/state")
738|async def get_game_state(game_id: str, current_user: dict = Depends(get_current_user)):
739|    username = current_user['username']
740|    game_key = f"{username}_{game_id}"
741|    
742|    if game_key not in active_games:
743|        raise HTTPException(status_code=404, detail="Game not found")
744|    
745|    game = active_games[game_key]
746|    return game.to_dict()
747|
748|# Include the router in the main app
749|app.include_router(api_router)
750|
751|app.add_middleware(
752|    CORSMiddleware,
753|    allow_credentials=True,
754|    allow_origins=["*"],
755|    allow_methods=["*"],
756|    allow_headers=["*"],
757|)
758|
759|# Configure logging
760|logging.basicConfig(
761|    level=logging.INFO,
762|    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
763|)
764|logger = logging.getLogger(__name__)
765|
766|@app.on_event("shutdown")
767|async def shutdown_db_client():
768|    client.close()
769|
