
# -*- coding: utf-8 -*-
"""PERCY JACKSON E O MAR DE MONSTROS - RPG de tabuleiro por turnos (projeto de fã, sem fins comerciais).
No VS Code: instale o Python 3.8+ e a extensão "Python", abra este arquivo e clique em ▶ (ou: python percy_mar_de_monstros.py).
Sem bibliotecas extras. O progresso é salvo em save_percy.json (mesma pasta)."""
import json, os, random, sys
from datetime import date, timedelta
 
try: sys.stdout.reconfigure(encoding="utf-8")
except Exception: pass
os.system("")  # ativa cores ANSI no Windows
 
SAVE = "save_percy.json"
ELEM = {"água": ["fogo"], "fogo": ["terra"], "terra": ["água", "raio"], "raio": ["água"]}  # a chave vence o valor (x1.5 de dano)
LEGENDA = "B=batalha E=evento T=tesouro P=pérola L=loja D=descanso X=chefe @=você ·=visitado"
PREMIOS = [(20, 0, 0, 0), (30, 0, 0, 0), (0, 3, 0, 0), (50, 0, 0, 0), (0, 0, 2, 0), (80, 0, 0, 0), (150, 2, 2, 1)]  # ciclo de 7 dias: dracmas, néctar, ambrosia, pontos de talento
HEROIS = {  # elemento, HP, PD (poder divino), ATK, habilidade (nome, custo, tipo, poder)
    "Percy": ("água", 100, 30, 14, ("Onda de Poseidon", 8, "dano", 1.9)),
    "Annabeth": ("terra", 85, 35, 12, ("Plano de Atena", 10, "buff", 0)),
    "Grover": ("terra", 80, 40, 8, ("Canção da Natureza", 9, "cura", 30)),
    "Tyson": ("fogo", 140, 20, 16, ("Martelada Ciclope", 10, "dano", 2.1)),
    "Clarisse": ("fogo", 120, 25, 15, ("Lança Elétrica", 9, "dano", 2.0)),
    "Thalia": ("raio", 110, 35, 18, ("Tempestade de Zeus", 10, "dano", 2.3)),
    "Arco-Íris": ("água", 90, 30, 11, ("Jato do Hipocampo", 8, "dano", 1.8)),
}
FALAS = {
    "Percy": ["Juntos a gente consegue!", "Já enfrentei coisa pior... acho."],
    "Annabeth": ["Água vence fogo, fogo vence terra, terra vence água. Usem isso!", "Mantenham a formação e mirem no ponto fraco!"],
    "Grover": ["Eu sinto o cheiro do medo... e é o meu!", "Se alguém se machucar, é só avisar!"],
    "Tyson": ["Tyson bate forte! Tyson protege o irmão!", "Monstro feio. Tyson gosta de bater em monstro feio."],
    "Clarisse": ["Saiam da frente, esse é meu!", "Se for pra cair, que seja com honra!"],
    "Thalia": ["Zeus está olhando. Vamos mostrar do que somos capazes!", "Segurem firme: vem raio aí!"],
    "Arco-Íris": ["Arco-Íris relincha: 'Pulem nas minhas costas, o mar é meu!'", "Arco-Íris faz um looping alegre na água."],
}
# (herói, fala, [(opção, efeito, valor, resposta)])
EVENTOS = [
    ("Annabeth", "Annabeth abre um mapa molhado: 'Dois caminhos: um calmo, outro cheio de corais brilhantes.'",
     [("Seguir o caminho calmo", "curar", 25, "Annabeth: 'Boa escolha. Dá tempo de respirar e planejar.'"),
      ("Arriscar os corais", "ouro", 40, "Annabeth: 'Tinha moedas! Mas me avisa antes de ser louco, ok?'")]),
    ("Tyson", "Tyson mostra uma concha enorme: 'Irmão, Tyson pode consertar o barco com isto?'",
     [("Deixar o Tyson tentar", "amizade", 2, "Tyson: 'Tyson feliz! Barco agora faz BUM bonito!'"),
      ("Pedir cautela", "ouro", 15, "Tyson: 'Tyson espera. Tyson é paciente... um pouquinho.'")]),
    ("Grover", "Grover treme: 'Sinto cheiro de monstro... ou de enchilada. Confia no meu nariz?'",
     [("Confiar no Grover", "amizade", 2, "Grover: 'Obrigado! Prometo que desta vez não é enchilada.'"),
      ("Revistar a área", "curar", 20, "Grover: 'Achei ervas! Pelo menos não era enchilada.'")]),
    ("Clarisse", "Clarisse aponta o horizonte: 'Vamos direto ou vamos treinar um pouco?'",
     [("Treinar juntos", "amizade", 2, "Clarisse: 'Boa. Mas não espere que eu pegue leve.'"),
      ("Seguir direto", "ouro", 30, "Clarisse: 'Assim que se fala! Mais rápido, mais butim.'")]),
]
# nome, cor ANSI, cenário, inimigos comuns, pérolas necessárias, chefe(s) (nome, HP, ATK, elemento, XP)
FASES = [
    {"nome": "Acampamento Meio-Sangue", "cor": 92, "arte": "🌲 ⛺ ⛺ 🏟  🌳☠ árvore de Thalia 🌲", "mobs": [], "perolas": 0, "chefe": None},
    {"nome": "Ruas de Nova York", "cor": 93, "arte": "🏙  🚕 🚖 🏢 🚕 🌆  (ponto de táxi das Graias)", "mobs": ["Ladrão de Sombras", "Cão Infernal", "Monstro de Beco"], "perolas": 2, "chefe": None},
    {"nome": "Mar dos Monstros", "cor": 96, "arte": "🌊 🌊 🚢 🌀 🦑 🌊 🌊", "mobs": ["Tubarão-Fantasma", "Sereia Venenosa", "Polvo de Coral"], "perolas": 3, "chefe": [("Caríbdis", 260, 18, "água", 120)]},
    {"nome": "Ilha de Polifemo", "cor": 33, "arte": "🏝  🐑 🐑 ⛰  👁  🐑 🏝", "mobs": ["Ovelha Selvagem", "Ciclope Guarda", "Touro de Bronze"], "perolas": 3, "chefe": [("Polifemo, o Ciclope", 380, 24, "terra", 180)]},
    {"nome": "Navio Princesa Andrômeda", "cor": 91, "arte": "⚓ ☠  🚢  ☠ ⚓", "mobs": ["Soldado de Cronos", "Harpia de Ferro", "Cavaleiro Sombrio"], "perolas": 3,
     "chefe": [("Luke Castellan", 420, 26, "sombra", 220), ("Monstro de Luke", 150, 20, "fogo", 60), ("Monstro de Luke", 150, 20, "terra", 60)]},
    {"nome": "Trono de Cronos", "cor": 95, "arte": "⏳ ⚡ 👑 ⚡ ⏳", "mobs": ["Servo do Tempo", "Sombra de Cronos"], "perolas": 3, "chefe": [("Cronos, Senhor do Tempo", 700, 34, "sombra", 400)]},
]
GRAIAS = [("Vespa", 30), ("Oídio", 50), ("Raiva", 80)]  # as 3 irmãs cinzentas, motoristas do táxi: preço em dracmas de ouro
PERKS = {"Vespa": "+20% de dracmas", "Oídio": "dado +1", "Raiva": "+2 ATK para todos"}
BRIGAS = ["Vespa: 'Me dá o dente! É a minha vez!'", "Oídio: 'O olho é MEU! Quero ver o dinheiro!'", "Raiva: 'Vocês duas são horríveis! Passa o dente pra cá!'"]
RARIDADES = {"comum": "⚪", "raro": "🔵", "épico": "🟣", "lendário": "🟡"}
RELIQUIAS = {  # nome: (raridade, atributo, valor, descrição) - bônus permanente para toda a equipe
    "Chifre do Touro de Bronze": ("comum", "hpmax", 10, "Troféu do seu primeiro monstro."),
    "Contas do Acampamento": ("comum", "hpmax", 12, "Cada conta marca um verão de treino."),
    "Pena de Pégaso": ("comum", "mpmax", 6, "Leve como o vento."),
    "Concha de Hipocampo": ("raro", "mpmax", 8, "Guarda o som do mar."),
    "Boné de Invisibilidade": ("raro", "atk", 2, "Quem o usa some do olhar dos monstros."),
    "Contracorrente": ("épico", "atk", 4, "A caneta que vira espada de bronze celestial."),
    "Caduceu de Hermes": ("épico", "mpmax", 15, "Presente do deus mensageiro."),
    "Dente e Olho das Graias": ("épico", "atk", 3, "Prêmio por convencer as três irmãs."),
    "Escama de Caríbdis": ("épico", "hpmax", 25, "Ainda cheira a maré."),
    "Tridente de Poseidon": ("lendário", "atk", 5, "O poder do senhor dos mares."),
    "Velocino de Ouro": ("lendário", "hpmax", 40, "Cura tudo o que toca."),
    "Espada Traiçoeira de Luke": ("lendário", "atk", 5, "Afiada, mas cheia de ressentimento."),
    "Foice de Cronos": ("lendário", "atk", 6, "O tempo corta quem a empunha."),
}
BOSS_RELIQUIA = {1: "Dente e Olho das Graias", 2: "Escama de Caríbdis", 3: "Velocino de Ouro", 4: "Espada Traiçoeira de Luke", 5: "Foice de Cronos"}
EMOJI = {"Percy": "🔱", "Annabeth": "🦉", "Grover": "🐐", "Tyson": "🔨", "Clarisse": "⚔", "Thalia": "⚡", "Arco-Íris": "🐴"}
CENAS_EXTRA = {"estomago": (32, "🦷 🦷 💦 🐟 🦴 💦 🦷 🦷", "Dentro de Caríbdis"), "meio": (36, "🌅 🌊 🌊 🐬 🌊 🌊 🌅", "No meio do caminho até a ilha"), "barco": (34, "⛵ 🌊 ⛵ 🌊 ⚓", "A bordo do barco")}
EXTRA_FASE = {3: "meio", 4: "barco"}
GUIA = [
    ("Tabuleiro e dado", "Role o dado e ande pelas casas. B=batalha, E=evento com a equipe, T=tesouro, P=Pérola de Poseidon, L=loja, D=descanso, X=chefe da fase."),
    ("Combate e elementos", "Cada herói age por turno: Atacar, Habilidade (gasta PD), Item ou Defender. Água vence fogo, fogo vence terra, terra vence água, e raio vence água: vantagem = +50% de dano."),
    ("Evolução", "Vitórias dão XP e dracmas. Cada nível dá 1 ponto de talento (Acampamento > Gastar pontos). Conversar no Lobby e escolher bem nos eventos aumenta o laço: a cada 3 pontos, +1 ATK."),
    ("Pérolas e fases", "Cada fase exige as pérolas (casas P) para abrir o chefe. Se chegar sem todas, o caminho se refaz e você tenta de novo, mantendo as pérolas."),
    ("Táxi das Graias", "Em Nova York, pague as irmãs Vespa, Oídio e Raiva em dracmas de ouro, ou arrisque ameaçar pegar o olho delas. Cada uma dá um bônus permanente."),
    ("Relíquias e Álbum", "Chefes, baús e vitórias dão relíquias (comum, raro, épico, lendário) com bônus permanentes para toda a equipe. Complete o Álbum no Lobby!"),
    ("Recompensas diárias", "Entre todo dia: sequência de 7 dias, missão de 3 vitórias, Bênção de Hermes e 1 conversa com cada herói. Tudo grátis."),
    ("Loja", "Usa só dracmas ganhos no jogo. A única compra com dinheiro real é um título cosmético, opcional."),
]
CENAS = ["Manhã no Acampamento Meio-Sangue. Percy treina na arena quando Quíron reúne todos: a árvore de Thalia foi envenenada e a barreira mágica está falhando.",
         "Tyson, o ciclope meio-irmão de Percy, chega ao acampamento. Annabeth descobre a única cura: o Velocino de Ouro, escondido no Mar dos Monstros.",
         "Percy decide partir com Annabeth, Grover e Tyson. Antes da fuga, o Touro de Bronze invade a arena!"]
DICAS = ["Annabeth: 'Comece com Atacar: é o golpe básico e não gasta PD.'",
         "Annabeth: 'Habilidade gasta PD, mas bate bem mais forte. Cada herói tem a sua!'",
         "Annabeth: 'Água vence fogo, fogo vence terra, terra vence água. O touro é de fogo: deixe o Percy brilhar!'",
         "Annabeth: 'Defender corta o dano pela metade e Item cura a equipe. Boa sorte, Percy!'"]
VIAGENS = ["",
    "🚶 À noite, vocês fogem do acampamento e chegam às ruas de Nova York. Para cruzar o mundo mortal, só um táxi muito especial...",
    "🚕 As três Graias pisam fundo e o asfalto vira onda: o táxi mergulha no Mar dos Monstros! Clarisse salta para dentro na última hora.",
    "🌊 Vocês escapam do redemoinho de Caríbdis e avistam uma ilha: ali vive o ciclope Polifemo, guardião do Velocino de Ouro.",
    "🐑 Com o Velocino em mãos, o navio Princesa Andrômeda cerca vocês: Luke quer o Velocino para despertar Cronos!",
    "⏳ Luke cai, mas o Velocino já despertou algo pior: o céu racha e Cronos, o Senhor do Tempo, surge diante de vocês!"]
FIM = "Cronos se desfaz no tempo e o Velocino de Ouro, enfim, cura a árvore: Thalia volta à vida, humana, meio-sangue e filha de Zeus! A barreira do acampamento brilha de novo.\n★ NOVO CICLO: os monstros voltam mais fortes!"
 
 
# ---------- utilidades ----------
def menu(titulo, opcoes):
    print(f"\n{titulo}")
    for i, o in enumerate(opcoes, 1): print(f"  {i}) {o}")
    while True:
        r = input("> ").strip()
        if r.isdigit() and 1 <= int(r) <= len(opcoes): return int(r) - 1
        print("Opção inválida.")
 
def vivos(l): return [x for x in l if x["hp"] > 0]
def mult(a, d): return 1.5 if d in ELEM.get(a, ()) else 0.75 if a in ELEM.get(d, ()) else 1.0
def fala(h): print(f"  💬 {h['nome']}: \"{random.choice(FALAS[h['nome']])}\"")
def salvar(s):
    with open(SAVE, "w", encoding="utf-8") as f: json.dump(s, f, ensure_ascii=False, indent=1)
 
def novo_heroi(nome):
    el, hp, mp, atk, hab = HEROIS[nome]
    return {"nome": nome, "el": el, "nivel": 1, "xp": 0, "hp": hp, "hpmax": hp, "mp": mp, "mpmax": mp, "atk": atk, "hab": list(hab), "pontos": 0, "amizade": 0}
 
def gerar_tab():
    t = ["·"] + random.choices("BBEPPTD", k=20) + ["X"]
    t[7], t[15] = "L", "D"
    return t
 
def novo_jogo():
    return {"equipe": [novo_heroi(n) for n in ("Percy", "Annabeth", "Grover", "Tyson")], "fase": 0, "perolas": 0, "graias": [], "album": [], "conv": {}, "pos": 0, "tab": gerar_tab(), "ciclo": 0,
            "dracmas": 50, "itens": {"Néctar": 3, "Ambrosia": 1}, "streak": 0, "ultimo_login": "", "missao": {}, "bencao": "", "apoiador": False}
 
def moldura(cor, arte, nome):
    print(f"\n\033[{cor}m{'═' * 56}\n   {arte}\n   {nome.upper()}\n{'═' * 56}\033[0m")
 
def cenario(i): moldura(FASES[i]["cor"], FASES[i]["arte"], FASES[i]["nome"])
def cena(k): moldura(*CENAS_EXTRA[k])
 
def aplicar_reliquia(h, nome):
    _, at, v, _ = RELIQUIAS[nome]
    h[at] += v
    if at in ("hpmax", "mpmax"): h[at[:2]] += v
 
def dar_reliquia(s, nome=None):
    if nome is None:
        fixas = set(BOSS_RELIQUIA.values()) | {"Chifre do Touro de Bronze"}
        livres = [n for n in RELIQUIAS if n not in s["album"] and n not in fixas]
        if not livres: return
        r = random.choices(["comum", "raro", "épico", "lendário"], [60, 28, 10, 2])[0]
        nome = random.choice([n for n in livres if RELIQUIAS[n][0] == r] or livres)
    if nome in s["album"]: return
    rar, at, v, desc = RELIQUIAS[nome]
    s["album"].append(nome)
    for h in s["equipe"]: aplicar_reliquia(h, nome)
    print(f"  ✨ {RARIDADES[rar]} RELÍQUIA {rar.upper()}: {nome} (+{v} {at.upper()} para todos) - {desc}")
 
def album(s):
    print(f"\n📖 ÁLBUM DE RELÍQUIAS ({len(s['album'])}/{len(RELIQUIAS)})")
    for n, (r, at, v, d) in RELIQUIAS.items():
        print(f"  {RARIDADES[r]} {n} (+{v} {at.upper()}) - {d}" if n in s["album"] else f"  ▫ ??? ({r})")
 
def guia():
    while True:
        i = menu("📘 Guia do jogo - escolha um tópico:", [g[0] for g in GUIA] + ["Voltar"])
        if i == len(GUIA): return
        print("\n" + GUIA[i][1])
 
def lobby(s):
    while True:
        cenario(s["fase"]); print("   " + "  ".join(EMOJI[h["nome"]] for h in s["equipe"]) + "   <- sua equipe")
        op = menu("Lobby:", ["Conversar com a equipe (1x por dia cada)", "Álbum de relíquias", "Guia do jogo", "Voltar ao tabuleiro"])
        if op == 0:
            h = s["equipe"][menu("Falar com:", [x["nome"] for x in s["equipe"]])]
            fala(h); hoje = date.today().isoformat()
            if s["conv"].get(h["nome"]) == hoje: print("  (Vocês já conversaram hoje.)"); continue
            s["conv"][h["nome"]] = hoje; h["amizade"] += 1
            if h["amizade"] % 3 == 0: h["atk"] += 1; print(f"  ❤ Laço com {h['nome']} mais forte! ATK +1")
            else: print(f"  ❤ Laço com {h['nome']}: {h['amizade']} (+1 ATK a cada 3)")
        elif op == 1: album(s)
        elif op == 2: guia()
        else: return
 
 
def mostrar(s):
    f = FASES[s["fase"]]
    print(f"\n=== Fase {s['fase']}/5: {f['nome']} | ciclo {s['ciclo'] + 1} | Pérolas {s['perolas']}/{f['perolas']} | Graias {len(s['graias'])}/3 ===")
    print("[ " + " ".join("@" if i == s["pos"] else t for i, t in enumerate(s["tab"])) + " ]\n" + LEGENDA)
    print(f"Dracmas: {s['dracmas']} | " + " | ".join(f"{h['nome']} Nv{h['nivel']} {h['hp']}/{h['hpmax']}" for h in s["equipe"]))
 
 
# ---------- evolução ----------
def subir(h):
    h["nivel"] += 1; h["hpmax"] += 12; h["mpmax"] += 3; h["atk"] += 2; h["pontos"] += 1
    h["hp"] = min(h["hpmax"], h["hp"] + 20)
 
def ganhar_xp(s, xp):
    for h in s["equipe"]:
        h["xp"] += xp
        while h["xp"] >= h["nivel"] * 40:
            h["xp"] -= h["nivel"] * 40; subir(h)
            print(f"  ★ {h['nome']} chegou ao nível {h['nivel']}! (+1 ponto de talento)")
 
def usar_item(s):
    itens = [k for k, v in s["itens"].items() if v > 0]
    if not itens: print("Você não tem itens."); return False
    i = itens[menu("Item:", [f"{k} x{s['itens'][k]}" for k in itens])]
    a = s["equipe"][menu("Em quem?", [f"{h['nome']} HP {h['hp']}/{h['hpmax']} PD {h['mp']}/{h['mpmax']}" for h in s["equipe"]])]
    if i == "Néctar": a["hp"] = min(a["hpmax"], max(a["hp"], 0) + 60)
    else: a["mp"] = min(a["mpmax"], a["mp"] + 25)
    s["itens"][i] -= 1; print(f"  {a['nome']} usou {i}."); return True
 
def acampamento(s):
    while True:
        op = menu("Acampamento", ["Ver equipe", "Gastar pontos de talento", "Usar item", "Voltar"])
        if op == 0:
            for h in s["equipe"]:
                print(f"  {h['nome']} Nv{h['nivel']} [{h['el']}] HP {h['hp']}/{h['hpmax']} PD {h['mp']}/{h['mpmax']} ATK {h['atk']} XP {h['xp']}/{h['nivel'] * 40} Pontos {h['pontos']} Laço {h['amizade']}")
        elif op == 1:
            h = s["equipe"][menu("Quem evolui?", [f"{x['nome']} ({x['pontos']} pontos)" for x in s["equipe"]])]
            if h["pontos"] < 1: print("Sem pontos de talento."); continue
            t = menu("Talento:", ["Força (+3 ATK)", "Vitalidade (+20 HP máx)", "Poder Divino (+10 PD máx)", "Maestria (melhora a habilidade)"])
            if t == 0: h["atk"] += 3
            elif t == 1: h["hpmax"] += 20; h["hp"] += 20
            elif t == 2: h["mpmax"] += 10; h["mp"] += 10
            else:
                tp = h["hab"][2]
                if tp == "buff": h["hab"][1] = max(3, h["hab"][1] - 2)
                else: h["hab"][3] += 0.2 if tp == "dano" else 6
            h["pontos"] -= 1; print("Talento aprendido!")
        elif op == 2: usar_item(s)
        else: return
 
 
# ---------- combate ----------
def golpe(h, e, poder, est):
    m = mult(h["el"], e["el"]); crit = random.random() < 0.12
    d = int(h["atk"] * poder * m * random.uniform(0.85, 1.15) * (1.3 if est["buff"] else 1) * (1.5 if crit else 1))
    e["hp"] -= d
    print(f"  ⚔ {h['nome']} causa {d} em {e['nome']}" + (" CRÍTICO!" if crit else "") + (" (super efetivo!)" if m > 1 else " (pouco efetivo)" if m < 1 else ""))
 
def ferir(e, a, p):
    d = e["atk"] * p * mult(e["el"], a["el"]) * random.uniform(0.85, 1.15)
    if a.get("def"): d *= 0.5
    d = int(d); a["hp"] -= d
    print(f"  💥 {e['nome']} causa {d} em {a['nome']} ({max(a['hp'], 0)}/{a['hpmax']})")
 
def alvo(inimigos):
    v = vivos(inimigos)
    return v[0] if len(v) == 1 else v[menu("Alvo:", [f"{e['nome']} [{e['el']}] {e['hp']} HP" for e in v])]
 
def turno_heroi(s, h, inimigos, est):
    nome_h, custo, tipo, poder = h["hab"]
    while True:
        op = menu("Ação:", ["Atacar", f"{nome_h} ({custo} PD)", "Usar item", "Defender"])
        if op == 0: golpe(h, alvo(inimigos), 1.0, est); return
        if op == 1:
            if h["mp"] < custo: print("Poder Divino insuficiente!"); continue
            h["mp"] -= custo
            if tipo == "dano": golpe(h, alvo(inimigos), poder, est)
            elif tipo == "cura":
                for a in vivos(s["equipe"]): a["hp"] = min(a["hpmax"], a["hp"] + int(poder) + a["nivel"] * 3)
                print(f"  🌿 {h['nome']} cura toda a equipe!")
            else: est["buff"] = 3; print(f"  📜 {h['nome']} traça um plano: dano da equipe +30% por 3 rodadas!")
            return
        if op == 2:
            if usar_item(s): return
            continue
        h["def"] = True; print(f"  🛡 {h['nome']} se defende."); return
 
def batalha(s, inimigos, tut=False):
    eq, est, rodada, dicas = s["equipe"], {"buff": 0}, 0, iter(DICAS if tut else [])
    print("\n⚔  Inimigos: " + ", ".join(f"{e['nome']} [{e['el']}]" for e in inimigos))
    while vivos(inimigos) and vivos(eq):
        rodada += 1
        for h in eq: h.pop("def", None)
        if random.random() < 0.4: fala(random.choice(vivos(eq)))
        for h in vivos(eq):
            if not vivos(inimigos): break
            print(f"\n--- Rodada {rodada} | " + " | ".join(f"{e['nome']} {e['hp']} HP" for e in vivos(inimigos)))
            print(f"Vez de {h['nome']}: HP {h['hp']}/{h['hpmax']}  PD {h['mp']}/{h['mpmax']}" + ("  [Plano ativo]" if est["buff"] else ""))
            d = next(dicas, None)
            if d: print("  🎓 " + d)
            turno_heroi(s, h, inimigos, est)
        for e in vivos(inimigos):
            if not vivos(eq): break
            if e.get("boss") and rodada % 3 == 0:
                print(f"  🌀 {e['nome']} usa o ataque devastador em TODOS!")
                for a in vivos(eq): ferir(e, a, 0.7)
            else: ferir(e, random.choice(vivos(eq)), 1.0)
        if est["buff"]: est["buff"] -= 1
    if not vivos(eq):
        print("\n💀 Derrota... Quíron resgata vocês e leva todos de volta ao início da zona.")
        s["dracmas"] = int(s["dracmas"] * 0.75); s["pos"] = 0
        for h in eq: h["hp"] = h["hpmax"] // 2
        return False
    xp, ouro = sum(e["xp"] for e in inimigos), int(sum(e["ouro"] for e in inimigos) * (1.2 if "Vespa" in s["graias"] else 1))
    print(f"\n🏆 Vitória! +{xp} XP e +{ouro} dracmas.")
    s["dracmas"] += ouro
    for h in eq: h["hp"] = max(h["hp"], h["hpmax"] // 4)
    ganhar_xp(s, xp)
    if random.random() < 0.3: print("  🎁 Achou 1 Néctar!"); s["itens"]["Néctar"] = s["itens"].get("Néctar", 0) + 1
    if random.random() < 0.12: dar_reliquia(s)
    m = s["missao"]; m["vitorias"] = m.get("vitorias", 0) + 1
    if m["vitorias"] == 3:
        s["dracmas"] += 60
        for h in eq: h["pontos"] += 1
        print("  📅 Missão diária concluída! +60 dracmas e +1 ponto de talento para todos.")
    return True
 
def gerar_inimigos(s):
    z, k = (s["fase"] - 1) * 0.7, 1 + 0.4 * s["ciclo"]
    return [{"nome": random.choice(FASES[s["fase"]]["mobs"]), "el": random.choice(list(ELEM)), "hp": int((45 + 25 * z + random.randint(0, 15)) * k),
             "atk": int((8 + 4 * z) * k), "xp": int(25 + 10 * z), "ouro": random.randint(10, 20) + int(8 * z)} for _ in range(random.choice([1, 2]))]
 
def recrutar(s, nome):
    if any(h["nome"] == nome for h in s["equipe"]): return
    c = novo_heroi(nome)
    for _ in range(round(sum(h["nivel"] for h in s["equipe"]) / len(s["equipe"])) - 1): subir(c)
    for n in s["album"]: aplicar_reliquia(c, n)
    c["atk"] += 2 * ("Raiva" in s["graias"])
    c["hp"], c["mp"] = c["hpmax"], c["mpmax"]; s["equipe"].append(c)
    print(f"  ⚔ {nome} entrou para a equipe!")
 
def avancar(s):
    s["fase"] += 1; s["pos"] = 0; s["perolas"] = 0; s["tab"] = gerar_tab()
    print("\n" + VIAGENS[s["fase"]])
    if s["fase"] in EXTRA_FASE: cena(EXTRA_FASE[s["fase"]])
    cenario(s["fase"])
    if s["fase"] == 2: recrutar(s, "Clarisse")
    if s["fase"] == 3: recrutar(s, "Arco-Íris")
 
def taxi(s):
    print("\n🚕 O táxi amarelo derrapa na calçada. Três velhinhas dividem UM olho e UM dente e só levam vocês ao Mar dos Monstros por dracmas de ouro!")
    while len(s["graias"]) < 3:
        print("  💬 " + random.choice(BRIGAS))
        livres = [(n, p) for n, p in GRAIAS if n not in s["graias"]]
        op = menu(f"Você tem {s['dracmas']} dracmas de ouro. Negociar com:", [f"{n} - {p} dracmas ({PERKS[n]})" for n, p in livres] + ["Sair e juntar mais dracmas"])
        if op == len(livres): return False
        n, p = livres[op]
        if menu(f"Como convencer {n}?", [f"Pagar {p} dracmas de ouro", "Ameaçar pegar o olho (arriscado)"]) == 0:
            if s["dracmas"] < p: print("Dracmas insuficientes."); continue
        elif random.random() < 0.35 + 0.04 * sum(h["nivel"] for h in s["equipe"]) / len(s["equipe"]):
            print(f"  👁 {n} grita: 'O OLHO! Tá bom, tá bom, entrem!' (de graça!)"); p = 0
        else:
            p = int(p * 1.2)
            for h in s["equipe"]: h["hp"] = max(1, h["hp"] - 15)
            print(f"  🦷 {n} morde e arranha a equipe (-15 HP) e a tarifa subiu para {p} dracmas.")
            if s["dracmas"] < p: continue
        s["dracmas"] -= p; s["graias"].append(n); print(f"  🚕 {n} assume o volante!")
        if n == "Raiva":
            for h in s["equipe"]: h["atk"] += 2
    return True
 
def chefe(s):
    f = FASES[s["fase"]]
    if s["perolas"] < f["perolas"]:
        print(f"\n🔒 O caminho está selado! Pérolas de Poseidon: {s['perolas']}/{f['perolas']}. Vocês voltam ao início para procurar mais.")
        s["pos"] = 0; s["tab"] = gerar_tab(); return
    if f["chefe"] is None:
        if not taxi(s): print("Vocês voltam às ruas para juntar mais dracmas."); s["pos"] = 0; s["tab"] = gerar_tab(); return
    else:
        k = 1 + 0.4 * s["ciclo"]
        if s["fase"] == 2: print("\n🌀 Caríbdis engole vocês de uma vez! Lutem por dentro do estômago!"); cena("estomago")
        bs = [{"nome": n, "el": el, "hp": int(hp * k), "atk": int(atk * k), "xp": xp, "ouro": 100 if i == 0 else 20, "boss": i == 0} for i, (n, hp, atk, el, xp) in enumerate(f["chefe"])]
        print("\n⚠  CHEFE: " + " + ".join(b["nome"] for b in bs) + "!")
        if not batalha(s, bs): return
    dar_reliquia(s, BOSS_RELIQUIA[s["fase"]])
    if s["fase"] == 5: final(s)
    else: avancar(s)
 
def final(s):
    print("\n" + FIM); recrutar(s, "Thalia")
    s["ciclo"] += 1; s["fase"] = 1; avancar(s)
 
def tutorial(s):
    cenario(0)
    for c in CENAS: print("\n" + c); input("[Enter]")
    print("\n🎓 TUTORIAL: o Touro de Bronze invade a arena! Annabeth vai te guiar.")
    while not batalha(s, [{"nome": "Touro de Bronze", "el": "fogo", "hp": 70, "atk": 7, "xp": 30, "ouro": 20}], True): print("Tente de novo!")
    dar_reliquia(s, "Chifre do Touro de Bronze"); print("💡 Dica: abra Lobby > Guia a qualquer momento para rever como o jogo funciona.")
    avancar(s)
 
 
# ---------- tabuleiro, eventos e loja ----------
def evento(s):
    nomes = [h["nome"] for h in s["equipe"]]
    nome, txt, ops = random.choice([e for e in EVENTOS if e[0] in nomes])
    h = next(x for x in s["equipe"] if x["nome"] == nome)
    print(f"\n💬 {txt}")
    _, tipo, v, resp = ops[menu("Sua resposta:", [o[0] for o in ops])]
    print("  " + resp)
    if tipo == "curar":
        for x in s["equipe"]: x["hp"] = min(x["hpmax"], x["hp"] + v)
        print(f"  Equipe recuperou {v} HP.")
    elif tipo == "ouro": s["dracmas"] += v; print(f"  +{v} dracmas.")
    else:
        h["amizade"] += v
        if h["amizade"] // 3 > (h["amizade"] - v) // 3: h["atk"] += 1; print(f"  ❤ Seu laço com {nome} ficou mais forte! ATK +1")
 
def apoiar(s):
    print("\n★ Pacote Apoiador (R$ 9,90): título 'Herói do Acampamento'. Só cosmético, não dá vantagem no jogo.")
    if input("Confirmar compra? (s/n) ").strip().lower() == "s":
        # AQUI entraria um pagamento real (Stripe, Google Play Billing...). No protótipo só liberamos o título.
        s["apoiador"] = True; print("Obrigado por apoiar o jogo! ⭐")
 
def loja(s):
    itens = [("Néctar", 15), ("Ambrosia", 20), ("Amuleto de Poseidon", 120)]
    while True:
        print(f"\n🏪 Mercador do Mar | você tem {s['dracmas']} dracmas")
        op = menu("Comprar:", ["Néctar (15) +60 HP", "Ambrosia (20) +25 PD", "Amuleto de Poseidon (120) +2 ATK permanente", "★ Pacote Apoiador (opcional, dinheiro real)", "Sair"])
        if op == 4: return
        if op == 3: apoiar(s); continue
        nome, preco = itens[op]
        if s["dracmas"] < preco: print("Dracmas insuficientes."); continue
        s["dracmas"] -= preco
        if op < 2: s["itens"][nome] = s["itens"].get(nome, 0) + 1
        else: s["equipe"][menu("Quem usa o amuleto?", [h["nome"] for h in s["equipe"]])]["atk"] += 2
        print("Comprado!")
 
def resolver(s):
    t = s["tab"][s["pos"]]
    if t == "B":
        if not batalha(s, gerar_inimigos(s)): return
    elif t == "T":
        q = random.randint(20, 50) + 8 * s["fase"]; s["dracmas"] += q; print(f"\n🎁 Baú do tesouro! +{q} dracmas")
        if random.random() < 0.4: s["itens"]["Ambrosia"] = s["itens"].get("Ambrosia", 0) + 1; print("  + 1 Ambrosia")
        if random.random() < 0.25: dar_reliquia(s)
    elif t == "P": s["perolas"] += 1; print(f"\n🐚 Pérola de Poseidon encontrada! ({s['perolas']}/{FASES[s['fase']]['perolas']})")
    elif t == "E": evento(s)
    elif t == "L": loja(s)
    elif t == "D":
        for h in s["equipe"]: h["hp"], h["mp"] = h["hpmax"], h["mpmax"]
        print("\n🏝 Uma ilhota tranquila. Todos descansam: HP e PD restaurados."); fala(random.choice(s["equipe"]))
    elif t == "X": chefe(s); return
    else: print("\nCaminho já explorado.")
    if t in "BETDP": s["tab"][s["pos"]] = "·"
 
 
# ---------- recompensas diárias (grátis) ----------
def recompensa_diaria(s):
    hoje = date.today()
    if s["ultimo_login"] == hoje.isoformat(): return
    s["streak"] = s["streak"] + 1 if s["ultimo_login"] == (hoje - timedelta(days=1)).isoformat() else 1
    s["ultimo_login"] = hoje.isoformat(); s["missao"] = {"vitorias": 0}
    dia = (s["streak"] - 1) % 7
    d, n, a, p = PREMIOS[dia]
    s["dracmas"] += d
    s["itens"]["Néctar"] = s["itens"].get("Néctar", 0) + n
    s["itens"]["Ambrosia"] = s["itens"].get("Ambrosia", 0) + a
    for h in s["equipe"]: h["pontos"] += p
    print(f"\n🎁 RECOMPENSA DIÁRIA - dia {dia + 1}/7 (sequência: {s['streak']})\n   +{d} dracmas, +{n} Néctar, +{a} Ambrosia" + (", +1 ponto de talento para todos" if p else ""))
 
def diario(s):
    hoje = date.today().isoformat(); m = s["missao"]
    print(f"\n📅 Sequência: {s['streak']} dia(s) | Missão de hoje: vencer 3 batalhas ({m.get('vitorias', 0)}/3)")
    if s["bencao"] == hoje: print("A Bênção de Hermes de hoje já foi usada. Volte amanhã!"); return
    if menu("Bênção grátis de Hermes disponível!", ["Receber", "Depois"]) == 0:
        r = random.choice(["dracmas", "Néctar", "Ambrosia"])
        if r == "dracmas": q = random.randint(40, 90); s["dracmas"] += q
        else: q = 2; s["itens"][r] = s["itens"].get(r, 0) + q
        s["bencao"] = hoje; print(f"Hermes entrega +{q} {r}!")
 
 
# ---------- jogo ----------
def main():
    print("=== PERCY JACKSON E O MAR DE MONSTROS - RPG de Tabuleiro ===")
    if os.path.exists(SAVE) and menu("Save encontrado:", ["Continuar", "Novo jogo"]) == 0:
        with open(SAVE, encoding="utf-8") as f: s = json.load(f)
        if "fase" not in s: print("Save da versão anterior: começando um novo jogo."); s = novo_jogo(); tutorial(s)
        else: s.setdefault("album", []); s.setdefault("conv", {}); s["graias"] = [g for g in s["graias"] if g in PERKS]; cenario(s["fase"])
    else:
        s = novo_jogo(); tutorial(s)
    recompensa_diaria(s)
    if s["apoiador"]: print("⭐ Título: Herói do Acampamento")
    while True:
        mostrar(s)
        op = menu("O que fazer?", ["Rolar o dado", "Acampamento (equipe, talentos, itens)", "Diário (missão e bênção diária)", "Lobby (cenário, conversar, álbum, guia)", "Salvar e sair"])
        if op == 0:
            d = random.randint(1, 6) + ("Oídio" in s["graias"]); s["pos"] = min(s["pos"] + d, len(s["tab"]) - 1)
            print(f"\n🎲 Você tirou {d}!"); resolver(s)
        elif op == 1: acampamento(s)
        elif op == 2: diario(s)
        elif op == 3: lobby(s)
        else: salvar(s); print("Jogo salvo. Até a próxima aventura!"); return
        salvar(s)
 
if __name__ == "__main__":
    main()