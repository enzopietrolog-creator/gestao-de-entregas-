"""
Módulo de relatório semanal de desempenho dos motoristas.
Gera relatório toda sexta-feira com ranking, estatísticas e apontamentos.
"""

import os
import json
from datetime import datetime, timedelta
from typing import List, Dict

from app.database import (
    get_ranking_motoristas,
    get_motivos_devolucao_periodo,
    get_connection,
    listar_motoristas,
)


def calcular_periodo_semana() -> tuple:
    """Calcula o período da semana atual (segunda a sexta)."""
    hoje = datetime.now()
    # Encontrar a segunda-feira desta semana
    segunda = hoje - timedelta(days=hoje.weekday())
    sexta = segunda + timedelta(days=4)
    return segunda.strftime("%d/%m/%Y"), sexta.strftime("%d/%m/%Y")


def calcular_periodo_semana_db() -> tuple:
    """Calcula o período da semana no formato do banco de dados."""
    hoje = datetime.now()
    segunda = hoje - timedelta(days=hoje.weekday())
    sexta = segunda + timedelta(days=4)
    return segunda.strftime("%Y-%m-%d"), sexta.strftime("%Y-%m-%d")


def get_dados_semana() -> dict:
    """Busca todos os dados da semana para o relatório."""
    data_inicio, data_fim = calcular_periodo_semana_db()
    conn = get_connection()

    # Total de cargas na semana
    total_cargas = conn.execute(
        "SELECT COUNT(*) as total FROM cargas WHERE data_carga >= ? AND data_carga <= ?",
        (data_inicio, data_fim),
    ).fetchone()["total"]

    # Total de entregas
    stats = conn.execute(
        """SELECT 
            COUNT(e.id) as total_entregas,
            SUM(CASE WHEN e.status = 'ENTREGUE' THEN 1 ELSE 0 END) as entregues,
            SUM(CASE WHEN e.status != 'ENTREGUE' THEN 1 ELSE 0 END) as pendentes
        FROM entregas e
        JOIN cargas c ON c.id = e.carga_id
        WHERE c.data_carga >= ? AND c.data_carga <= ?""",
        (data_inicio, data_fim),
    ).fetchone()

    # Ranking de motoristas
    ranking = conn.execute(
        """SELECT 
            m.codigo,
            m.nome,
            COUNT(DISTINCT c.id) as total_cargas,
            COUNT(e.id) as total_entregas,
            SUM(CASE WHEN e.status = 'ENTREGUE' THEN 1 ELSE 0 END) as entregues,
            SUM(CASE WHEN e.status != 'ENTREGUE' THEN 1 ELSE 0 END) as nao_entregues
        FROM motoristas m
        JOIN cargas c ON c.motorista_codigo = m.codigo
        JOIN entregas e ON e.carga_id = c.id
        WHERE c.data_carga >= ? AND c.data_carga <= ?
        GROUP BY m.codigo
        ORDER BY (CAST(SUM(CASE WHEN e.status = 'ENTREGUE' THEN 1 ELSE 0 END) AS FLOAT) / MAX(COUNT(e.id), 1)) DESC""",
        (data_inicio, data_fim),
    ).fetchall()

    # Motivos de devolução
    motivos = conn.execute(
        """SELECT e.motivo_devolucao, COUNT(*) as quantidade
        FROM entregas e
        JOIN cargas c ON c.id = e.carga_id
        WHERE c.data_carga >= ? AND c.data_carga <= ?
        AND e.motivo_devolucao IS NOT NULL AND e.motivo_devolucao != ''
        GROUP BY e.motivo_devolucao
        ORDER BY quantidade DESC""",
        (data_inicio, data_fim),
    ).fetchall()

    # Notas sem motivo registrado
    sem_motivo = conn.execute(
        """SELECT COUNT(*) as total
        FROM entregas e
        JOIN cargas c ON c.id = e.carga_id
        WHERE c.data_carga >= ? AND c.data_carga <= ?
        AND e.status != 'ENTREGUE'
        AND (e.motivo_devolucao IS NULL OR e.motivo_devolucao = '')""",
        (data_inicio, data_fim),
    ).fetchone()["total"]

    conn.close()

    return {
        "total_cargas": total_cargas,
        "total_entregas": stats["total_entregas"] or 0,
        "entregues": stats["entregues"] or 0,
        "pendentes": stats["pendentes"] or 0,
        "ranking": [dict(r) for r in ranking],
        "motivos": [dict(m) for m in motivos],
        "sem_motivo": sem_motivo,
    }


def gerar_medalha(posicao: int) -> str:
    """Retorna emoji de medalha baseado na posição."""
    medalhas = {1: "🥇", 2: "🥈", 3: "🥉"}
    return medalhas.get(posicao, f"{posicao}º")


def gerar_barra_progresso(percentual: float) -> str:
    """Gera barra de progresso visual."""
    preenchido = int(percentual / 10)
    vazio = 10 - preenchido
    return "▓" * preenchido + "░" * vazio


def gerar_relatorio_semanal() -> str:
    """Gera o relatório semanal completo formatado para WhatsApp."""
    periodo_inicio, periodo_fim = calcular_periodo_semana()
    dados = get_dados_semana()

    total = max(dados["total_entregas"], 1)
    taxa_geral = round(dados["entregues"] / total * 100, 1)

    texto = f"""📊 *RELATÓRIO SEMANAL DE DESEMPENHO*
━━━━━━━━━━━━━━━━━━━━━━━━━━
📅 Período: *{periodo_inicio}* a *{periodo_fim}*
━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 *RESUMO GERAL*
┌─────────────────────────
│ 🚛 Total de cargas: *{dados['total_cargas']}*
│ 📋 Total de entregas: *{dados['total_entregas']}*
│ ✅ Realizadas: *{dados['entregues']}*
│ ❌ Não entregues: *{dados['pendentes']}*
│ 📊 Taxa de entrega: *{taxa_geral}%*
│ {gerar_barra_progresso(taxa_geral)} {taxa_geral}%
└─────────────────────────
"""

    # Ranking de motoristas
    if dados["ranking"]:
        texto += """
🏆 *RANKING DE MOTORISTAS*
━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
        for i, m in enumerate(dados["ranking"], 1):
            total_m = max(m["total_entregas"], 1)
            taxa = round(m["entregues"] / total_m * 100, 1)
            medalha = gerar_medalha(i)
            barra = gerar_barra_progresso(taxa)

            texto += f"""
{medalha} *{m['nome']}*
   📦 Entregas: {m['entregues']}/{m['total_entregas']} | 🚛 Cargas: {m['total_cargas']}
   {barra} *{taxa}%*
   ❌ Não entregues: {m['nao_entregues']}
"""

    # Motivos de devolução
    if dados["motivos"]:
        texto += """
━━━━━━━━━━━━━━━━━━━━━━━━━━
📝 *MOTIVOS DE DEVOLUÇÃO*
"""
        for m in dados["motivos"]:
            texto += f"   • {m['motivo_devolucao']}: *{m['quantidade']}* ocorrência(s)\n"

    if dados["sem_motivo"] > 0:
        texto += f"""
⚠️ *ATENÇÃO:* {dados['sem_motivo']} nota(s) sem motivo de devolução registrado!
"""

    # Destaques
    if dados["ranking"]:
        melhor = dados["ranking"][0]
        total_melhor = max(melhor["total_entregas"], 1)
        taxa_melhor = round(melhor["entregues"] / total_melhor * 100, 1)

        texto += f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ *DESTAQUE DA SEMANA*
🏆 Melhor desempenho: *{melhor['nome']}* com *{taxa_melhor}%* de entregas realizadas!
"""

    texto += """
━━━━━━━━━━━━━━━━━━━━━━━━━━
_Relatório gerado automaticamente pelo Sistema de Controle de Entregas_"""

    return texto


def gerar_relatorio_motorista(motorista_codigo: str) -> str:
    """Gera relatório individual de um motorista."""
    data_inicio, data_fim = calcular_periodo_semana_db()
    conn = get_connection()

    motorista = conn.execute(
        "SELECT * FROM motoristas WHERE codigo = ?", (motorista_codigo,)
    ).fetchone()

    if not motorista:
        return "❌ Motorista não encontrado."

    # Estatísticas do motorista
    stats = conn.execute(
        """SELECT 
            COUNT(DISTINCT c.id) as total_cargas,
            COUNT(e.id) as total_entregas,
            SUM(CASE WHEN e.status = 'ENTREGUE' THEN 1 ELSE 0 END) as entregues,
            SUM(CASE WHEN e.status != 'ENTREGUE' THEN 1 ELSE 0 END) as pendentes
        FROM cargas c
        JOIN entregas e ON e.carga_id = c.id
        WHERE c.motorista_codigo = ? AND c.data_carga >= ? AND c.data_carga <= ?""",
        (motorista_codigo, data_inicio, data_fim),
    ).fetchone()

    # Detalhes das devoluções
    devolucoes = conn.execute(
        """SELECT e.nf_numero, e.cliente_nome, e.cliente_fantasia, e.motivo_devolucao
        FROM entregas e
        JOIN cargas c ON c.id = e.carga_id
        WHERE c.motorista_codigo = ? AND c.data_carga >= ? AND c.data_carga <= ?
        AND e.status != 'ENTREGUE'""",
        (motorista_codigo, data_inicio, data_fim),
    ).fetchall()

    conn.close()

    total = max(stats["total_entregas"] or 1, 1)
    entregues = stats["entregues"] or 0
    taxa = round(entregues / total * 100, 1)
    periodo_inicio, periodo_fim = calcular_periodo_semana()

    texto = f"""📋 *RELATÓRIO INDIVIDUAL*
━━━━━━━━━━━━━━━━━━━━━━━━━━
👤 Motorista: *{motorista['nome']}*
📅 Período: {periodo_inicio} a {periodo_fim}
━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 *DESEMPENHO*
🚛 Cargas: *{stats['total_cargas']}*
📋 Total de entregas: *{stats['total_entregas']}*
✅ Realizadas: *{entregues}*
❌ Não entregues: *{stats['pendentes']}*
📊 Taxa: *{taxa}%*
{gerar_barra_progresso(taxa)} {taxa}%
"""

    if devolucoes:
        texto += "\n📝 *DEVOLUÇÕES:*\n"
        for d in devolucoes:
            nome = d["cliente_fantasia"] or d["cliente_nome"]
            motivo = d["motivo_devolucao"] or "Sem motivo"
            texto += f"   ❌ NF-e {d['nf_numero']} — {nome}\n      Motivo: {motivo}\n"

    texto += "\n_Relatório gerado pelo Sistema de Controle de Entregas_"
    return texto


if __name__ == "__main__":
    # Teste
    relatorio = gerar_relatorio_semanal()
    print(relatorio)
    
Módulo de cruzamento de dados.
Cruza os dados da carga (PDF) com os dados das canhoteiras (imagens)
e gera o relatório de entregas realizadas vs pendentes.
Também gera as mensagens de cobrança automática.
"""

import json
from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class ResultadoEntrega:
    nf_numero: str
    cliente_codigo: str
    cliente_nome: str
    cliente_fantasia: str
    cnpj: str
    endereco: str
    telefone: str
    total_volumes: int
    produtos: List[dict]
    # Dados da canhoteira
    status: str  # "ENTREGUE", "PENDENTE", "NAO_ENCONTRADA"
    data_recebimento: str = ""
    recebedor_nome: str = ""
    recebedor_documento: str = ""
    temperatura: str = ""
    valor: str = ""
    motivo_devolucao: str = ""
    aguardando_motivo: bool = False


def normalizar_nf(nf: str) -> str:
    """Normaliza número de NF-e removendo caracteres não numéricos."""
    return "".join(c for c in str(nf) if c.isdigit())


def cruzar_dados(carga_dict: dict, canhoteiras: List[dict]) -> List[ResultadoEntrega]:
    """
    Cruza os dados da carga com os dados das canhoteiras analisadas.
    Retorna lista de ResultadoEntrega com o status de cada nota.
    """
    resultados = []

    # Criar mapa de canhoteiras por NF-e (normalizado)
    mapa_canhoteiras = {}
    for c in canhoteiras:
        nf_norm = normalizar_nf(c.get("nf_numero", ""))
        if nf_norm:
            mapa_canhoteiras[nf_norm] = c

    # Para cada nota da carga, verificar se tem canhoteira correspondente
    for nota in carga_dict.get("notas", []):
        nf_norm = normalizar_nf(nota.get("numero", ""))

        resultado = ResultadoEntrega(
            nf_numero=nota.get("numero", ""),
            cliente_codigo=nota.get("cliente_codigo", ""),
            cliente_nome=nota.get("cliente_nome", ""),
            cliente_fantasia=nota.get("cliente_fantasia", ""),
            cnpj=nota.get("cnpj", ""),
            endereco=f"{nota.get('endereco', '')} - {nota.get('bairro', '')}",
            telefone=nota.get("telefone", ""),
            total_volumes=nota.get("total_volumes", 0),
            produtos=nota.get("produtos", []),
            status="NAO_ENCONTRADA",
        )

        # Procurar canhoteira correspondente
        canhoteira = mapa_canhoteiras.get(nf_norm)

        if canhoteira:
            resultado.status = canhoteira.get("status", "PENDENTE")
            resultado.data_recebimento = canhoteira.get("data_recebimento", "")
            resultado.recebedor_nome = canhoteira.get("recebedor_nome", "")
            resultado.recebedor_documento = canhoteira.get("recebedor_documento", "")
            resultado.temperatura = canhoteira.get("temperatura", "")
            resultado.valor = canhoteira.get("valor", "")

            if resultado.status == "PENDENTE":
                resultado.aguardando_motivo = True
        else:
            # Nota não encontrada nas canhoteiras - pode ser pendente
            resultado.status = "NAO_ENCONTRADA"
            resultado.aguardando_motivo = True

        resultados.append(resultado)

    return resultados


def gerar_resumo_texto(resultados: List[ResultadoEntrega], carga_dict: dict) -> str:
    """
    Gera um resumo em texto formatado para enviar no WhatsApp.
    """
    carregamento = carga_dict.get("numero_carregamento", "")
    motorista = carga_dict.get("motorista_nome", "")
    data = carga_dict.get("data", "")

    entregues = [r for r in resultados if r.status == "ENTREGUE"]
    pendentes = [r for r in resultados if r.status in ("PENDENTE", "NAO_ENCONTRADA")]

    texto = f"""📦 *RELATÓRIO DE ENTREGAS*
━━━━━━━━━━━━━━━━━━━━
🚛 Carregamento: *{carregamento}*
👤 Motorista: *{motorista}*
📅 Data: *{data}*
━━━━━━━━━━━━━━━━━━━━

✅ *ENTREGAS REALIZADAS: {len(entregues)}/{len(resultados)}*
"""

    for r in entregues:
        nome_display = r.cliente_fantasia if r.cliente_fantasia else r.cliente_nome
        texto += f"""
📋 NF-e *{r.nf_numero}* — {nome_display}
   📝 Recebedor: {r.recebedor_nome}
   📦 Volumes: {r.total_volumes} | 🌡️ Temp: {r.temperatura}°C
"""

    if pendentes:
        texto += f"""
━━━━━━━━━━━━━━━━━━━━
⚠️ *ENTREGAS PENDENTES: {len(pendentes)}/{len(resultados)}*
"""
        for r in pendentes:
            nome_display = r.cliente_fantasia if r.cliente_fantasia else r.cliente_nome
            texto += f"""
❌ NF-e *{r.nf_numero}* — {nome_display}
   📍 {r.endereco}
   📦 Volumes: {r.total_volumes}
"""

    texto += f"""
━━━━━━━━━━━━━━━━━━━━
📊 *TAXA DE ENTREGA: {len(entregues)}/{len(resultados)} ({len(entregues)*100//max(len(resultados),1)}%)*
"""

    return texto


def gerar_mensagem_cobranca(resultado: ResultadoEntrega, numero_carga: str) -> str:
    """
    Gera mensagem de cobrança automática para nota pendente.
    """
    nome_display = resultado.cliente_fantasia if resultado.cliente_fantasia else resultado.cliente_nome

    texto = f"""⚠️ *ATENÇÃO — NOTA SEM ASSINATURA / NÃO ENTREGUE*
━━━━━━━━━━━━━━━━━━━━
🚛 Carga: *{numero_carga}*
📋 NF-e: *{resultado.nf_numero}*
🏪 Cliente: *{nome_display}*
📍 Endereço: {resultado.endereco}
📦 Volumes: {resultado.total_volumes}
━━━━━━━━━━━━━━━━━━━━

❓ *Por favor, informe o motivo:*

1️⃣ Cliente fechado / ausente
2️⃣ Cliente recusou a mercadoria
3️⃣ Endereço não encontrado
4️⃣ Mercadoria com avaria
5️⃣ Falta de produto no carregamento
6️⃣ Erro de roteirização
7️⃣ Problema com o veículo
8️⃣ Outro motivo

_Responda com o número da opção ou descreva o motivo._"""

    return texto


MOTIVOS_DEVOLUCAO = {
    "1": "Cliente fechado / ausente",
    "2": "Cliente recusou a mercadoria",
    "3": "Endereço não encontrado",
    "4": "Mercadoria com avaria",
    "5": "Falta de produto no carregamento",
    "6": "Erro de roteirização",
    "7": "Problema com o veículo",
    "8": "Outro motivo",
}


def interpretar_motivo(resposta: str) -> str:
    """
    Interpreta a resposta do motorista sobre o motivo da devolução.
    """
    resposta = resposta.strip()

    # Verificar se é um número de opção
    if resposta in MOTIVOS_DEVOLUCAO:
        return MOTIVOS_DEVOLUCAO[resposta]

    # Se não é número, usar o texto como motivo livre
    return f"Outro: {resposta}"


def gerar_confirmacao_motivo(nf_numero: str, motivo: str) -> str:
    """
    Gera mensagem de confirmação após receber o motivo.
    """
    return f"""✅ *Motivo registrado com sucesso!*

📋 NF-e: *{nf_numero}*
📝 Motivo: *{motivo}*

_Informação registrada no sistema._"""


def resultado_para_dict(resultados: List[ResultadoEntrega]) -> List[dict]:
    """Converte resultados para lista de dicionários."""
    return [
        {
            "nf_numero": r.nf_numero,
            "cliente_codigo": r.cliente_codigo,
            "cliente_nome": r.cliente_nome,
            "cliente_fantasia": r.cliente_fantasia,
            "cnpj": r.cnpj,
            "endereco": r.endereco,
            "telefone": r.telefone,
            "total_volumes": r.total_volumes,
            "status": r.status,
            "data_recebimento": r.data_recebimento,
            "recebedor_nome": r.recebedor_nome,
            "recebedor_documento": r.recebedor_documento,
            "temperatura": r.temperatura,
            "valor": r.valor,
            "motivo_devolucao": r.motivo_devolucao,
            "aguardando_motivo": r.aguardando_motivo,
        }
        for r in resultados
    ]


if __name__ == "__main__":
    # Teste com dados de exemplo
    with open("/home/ubuntu/upload/LUISE.P-28_carga.json", "r") as f:
        carga = json.load(f)

    # Simular canhoteiras
    canhoteiras_exemplo = [
        {"nf_numero": "383458", "status": "ENTREGUE", "recebedor_nome": "Rubens Santos",
         "data_recebimento": "27/02/26", "temperatura": "5", "valor": "R$2.602,00"},
        {"nf_numero": "383459", "status": "ENTREGUE", "recebedor_nome": "Ana",
         "data_recebimento": "27/02/26", "temperatura": "6", "valor": "R$403,70"},
        {"nf_numero": "383463", "status": "PENDENTE", "recebedor_nome": "",
         "data_recebimento": "", "temperatura": "", "valor": "R$697,40"},
    ]

    resultados = cruzar_dados(carga, canhoteiras_exemplo)
    resumo = gerar_resumo_texto(resultados, carga)
    print(resumo)

    pendentes = [r for r in resultados if r.aguardando_motivo]
    for p in pendentes:
        msg = gerar_mensagem_cobranca(p, carga["numero_carregamento"])
        print(f"\n{msg}\n")
"""
Módulo de análise de imagens de canhoteiras.
Usa a API OpenAI (gpt-4.1-mini com visão) para analisar fotos de canhoteiras
e extrair informações de cada bloco: NF-e, assinatura, data, volumes, temperatura.
"""

import os
import json
import base64
import re
from typing import List, Optional
from dataclasses import dataclass, field
from openai import OpenAI


@dataclass
class CanhoteiraParsed:
    nf_numero: str
    cliente_nome: str
    valor: str
    data_recebimento: str
    recebedor_nome: str
    recebedor_documento: str
    volumes: str
    temperatura: str
    tem_assinatura: bool
    status: str  # "ENTREGUE" ou "PENDENTE"
    observacao: str = ""


def encode_image_base64(image_path: str) -> str:
    """Converte imagem para base64."""
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")


def analisar_canhoteira(image_path: str) -> List[CanhoteiraParsed]:
    """
    Analisa uma foto de canhoteira usando GPT-4.1-mini com visão.
    Retorna lista de canhoteiras identificadas na imagem.
    """
    client = OpenAI()

    base64_image = encode_image_base64(image_path)

    # Determinar tipo MIME
    ext = os.path.splitext(image_path)[1].lower()
    mime_map = {".jpg": "image/jpeg", ".jpeg": "image/jpeg", ".png": "image/png"}
    mime_type = mime_map.get(ext, "image/jpeg")

    prompt = """Analise esta foto de um relatório de entrega (canhoteiras) da empresa Speers Foods.

Para CADA bloco/canhoteira visível na imagem, extraia as seguintes informações:

1. **nf_numero**: Número da NF-e (ex: 383458)
2. **cliente_nome**: Nome do cliente/destinatário impresso
3. **valor**: Valor em R$ impresso
4. **data_recebimento**: Data manuscrita de recebimento (se houver). Se não houver data manuscrita, retorne ""
5. **recebedor_nome**: Nome manuscrito do recebedor (se houver). Se não houver nome manuscrito, retorne ""
6. **recebedor_documento**: Número de documento manuscrito do recebedor (se houver). Se não houver, retorne ""
7. **volumes**: Número de volumes manuscrito
8. **temperatura**: Temperatura manuscrita em °C
9. **tem_assinatura**: true se há assinatura/nome manuscrito visível, false se o campo está em branco
10. **status**: "ENTREGUE" se tem assinatura/dados manuscritos, "PENDENTE" se os campos manuscritos estão em branco

IMPORTANTE:
- Campos IMPRESSOS (tipografia) são dados da nota fiscal
- Campos MANUSCRITOS (caneta azul/preta) são dados preenchidos na entrega
- Se um bloco tem campos manuscritos em branco (sem data, sem nome, sem assinatura), o status é "PENDENTE"
- Se um bloco tem dados manuscritos preenchidos, o status é "ENTREGUE"

Retorne APENAS um JSON válido no formato:
{
  "canhoteiras": [
    {
      "nf_numero": "383458",
      "cliente_nome": "SUPERMERCADO KATUCHA",
      "valor": "R$2.602,00",
      "data_recebimento": "27/02/26",
      "recebedor_nome": "Rubens Santos",
      "recebedor_documento": "09.942.315/0002-03",
      "volumes": "15",
      "temperatura": "5",
      "tem_assinatura": true,
      "status": "ENTREGUE"
    }
  ]
}

Analise TODOS os blocos visíveis na imagem, de cima para baixo."""

    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:{mime_type};base64,{base64_image}",
                            "detail": "high",
                        },
                    },
                ],
            }
        ],
        max_tokens=4000,
        temperature=0.1,
    )

    # Extrair JSON da resposta
    content = response.choices[0].message.content
    
    # Tentar extrair JSON do texto
    json_match = re.search(r"\{[\s\S]*\}", content)
    if not json_match:
        print(f"Erro: Não foi possível extrair JSON da resposta")
        print(f"Resposta: {content}")
        return []

    try:
        data = json.loads(json_match.group())
    except json.JSONDecodeError as e:
        print(f"Erro ao decodificar JSON: {e}")
        print(f"Texto: {json_match.group()[:500]}")
        return []

    canhoteiras = []
    for item in data.get("canhoteiras", []):
        c = CanhoteiraParsed(
            nf_numero=str(item.get("nf_numero", "")),
            cliente_nome=str(item.get("cliente_nome", "")),
            valor=str(item.get("valor", "")),
            data_recebimento=str(item.get("data_recebimento", "")),
            recebedor_nome=str(item.get("recebedor_nome", "")),
            recebedor_documento=str(item.get("recebedor_documento", "")),
            volumes=str(item.get("volumes", "")),
            temperatura=str(item.get("temperatura", "")),
            tem_assinatura=bool(item.get("tem_assinatura", False)),
            status=str(item.get("status", "PENDENTE")),
        )
        canhoteiras.append(c)

    return canhoteiras


def analisar_multiplas_imagens(image_paths: List[str]) -> List[CanhoteiraParsed]:
    """
    Analisa múltiplas imagens de canhoteiras e retorna todas as canhoteiras encontradas.
    """
    todas_canhoteiras = []
    for path in image_paths:
        print(f"Analisando: {path}")
        canhoteiras = analisar_canhoteira(path)
        print(f"  → {len(canhoteiras)} canhoteiras encontradas")
        todas_canhoteiras.extend(canhoteiras)
    return todas_canhoteiras


def resultado_para_dict(canhoteiras: List[CanhoteiraParsed]) -> List[dict]:
    """Converte lista de canhoteiras para lista de dicionários."""
    return [
        {
            "nf_numero": c.nf_numero,
            "cliente_nome": c.cliente_nome,
            "valor": c.valor,
            "data_recebimento": c.data_recebimento,
            "recebedor_nome": c.recebedor_nome,
            "recebedor_documento": c.recebedor_documento,
            "volumes": c.volumes,
            "temperatura": c.temperatura,
            "tem_assinatura": c.tem_assinatura,
            "status": c.status,
        }
        for c in canhoteiras
    ]


if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Uso: python image_analyzer.py <imagem1> [imagem2] ...")
        sys.exit(1)

    images = sys.argv[1:]
    canhoteiras = analisar_multiplas_imagens(images)

    print(f"\n{'='*60}")
    print(f"TOTAL DE CANHOTEIRAS ANALISADAS: {len(canhoteiras)}")
    print(f"{'='*60}\n")

    entregues = [c for c in canhoteiras if c.status == "ENTREGUE"]
    pendentes = [c for c in canhoteiras if c.status == "PENDENTE"]

    print(f"✅ ENTREGUES: {len(entregues)}")
    for c in entregues:
        print(f"   NF-e {c.nf_numero} | {c.cliente_nome} | {c.recebedor_nome}")

    print(f"\n❌ PENDENTES: {len(pendentes)}")
    for c in pendentes:
        print(f"   NF-e {c.nf_numero} | {c.cliente_nome}")
"""
Módulo de extração de dados de PDFs de carga.
Lê PDFs no formato Speers Foods e extrai todas as notas fiscais,
clientes, volumes, valores e produtos.
"""

import re
import subprocess
import json
import os
from dataclasses import dataclass, field, asdict
from typing import List, Optional


@dataclass
class Produto:
    codigo: str
    descricao: str
    volumes: int
    quantidade: Optional[int] = None


@dataclass
class NotaFiscal:
    numero: str
    cliente_codigo: str
    cliente_nome: str
    cliente_fantasia: str
    cnpj: str
    endereco: str
    bairro: str
    cep: str
    telefone: str
    data_entrega: str
    produtos: List[Produto] = field(default_factory=list)
    total_volumes: int = 0


@dataclass
class Carga:
    numero_carregamento: str
    motorista_codigo: str
    motorista_nome: str
    data: str
    notas: List[NotaFiscal] = field(default_factory=list)

    def to_dict(self):
        return {
            "numero_carregamento": self.numero_carregamento,
            "motorista_codigo": self.motorista_codigo,
            "motorista_nome": self.motorista_nome,
            "data": self.data,
            "notas": [
                {
                    "numero": n.numero,
                    "cliente_codigo": n.cliente_codigo,
                    "cliente_nome": n.cliente_nome,
                    "cliente_fantasia": n.cliente_fantasia,
                    "cnpj": n.cnpj,
                    "endereco": n.endereco,
                    "bairro": n.bairro,
                    "cep": n.cep,
                    "telefone": n.telefone,
                    "data_entrega": n.data_entrega,
                    "total_volumes": n.total_volumes,
                    "produtos": [
                        {
                            "codigo": p.codigo,
                            "descricao": p.descricao,
                            "volumes": p.volumes,
                            "quantidade": p.quantidade,
                        }
                        for p in n.produtos
                    ],
                }
                for n in self.notas
            ],
        }


def extrair_texto_pdf(pdf_path: str) -> str:
    """Extrai texto do PDF usando pdftotext."""
    try:
        result = subprocess.run(
            ["pdftotext", "-layout", pdf_path, "-"],
            capture_output=True,
            text=True,
            timeout=30,
        )
        return result.stdout
    except Exception as e:
        print(f"Erro ao extrair texto do PDF: {e}")
        return ""


def parse_carga_pdf(pdf_path: str) -> Carga:
    """
    Faz o parse completo de um PDF de carga no formato Speers Foods.
    Retorna um objeto Carga com todas as notas fiscais e produtos.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair número do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)

        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()

        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Encontrar blocos de clientes
    i = 0
    while i < len(linhas):
        linha = linhas[i]

        # Detectar início de bloco de cliente
        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", linha)
        if m_cliente:
            nota = NotaFiscal(
                numero="",
                cliente_codigo=m_cliente.group(1).strip(),
                cliente_nome=m_cliente.group(2).strip(),
                cliente_fantasia="",
                cnpj="",
                endereco="",
                bairro="",
                cep="",
                telefone="",
                data_entrega="",
            )

            # Procurar dados do cliente nas próximas linhas
            j = i + 1
            while j < min(i + 15, len(linhas)):
                l = linhas[j]

                # CNPJ e Fantasia
                m_cnpj = re.search(
                    r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", l
                )
                if m_cnpj:
                    nota.cnpj = m_cnpj.group(1).strip()
                    nota.cliente_fantasia = m_cnpj.group(2).strip()

                # Endereço
                m_end = re.search(
                    r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
                    l,
                )
                if m_end:
                    nota.endereco = m_end.group(1).strip()
                    nota.bairro = m_end.group(3).strip()

                # Data de entrega, telefone, CEP
                m_data = re.search(r"Data da entrega:\s*(\S+)", l)
                if m_data:
                    nota.data_entrega = m_data.group(1).strip()
                    if not carga.data:
                        carga.data = nota.data_entrega

                m_tel = re.search(r"Telefone:\s*(\S+)", l)
                if m_tel:
                    nota.telefone = m_tel.group(1).strip()

                m_cep = re.search(r"CEP:\s*(\d+)", l)
                if m_cep:
                    nota.cep = m_cep.group(1).strip()

                # Nota fiscal
                m_nota = re.search(r"NOTA:\s*(\d+)", l)
                if m_nota:
                    nota.numero = m_nota.group(1).strip()

                # Produtos (linhas com código numérico no início)
                m_prod = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s*$", l
                )
                if m_prod:
                    prod = Produto(
                        codigo=m_prod.group(1).strip(),
                        descricao=m_prod.group(2).strip(),
                        volumes=int(m_prod.group(3).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Produto com quantidade entregue
                m_prod2 = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s+(\d+)\s*$", l
                )
                if m_prod2 and not m_prod:
                    prod = Produto(
                        codigo=m_prod2.group(1).strip(),
                        descricao=m_prod2.group(2).strip(),
                        volumes=int(m_prod2.group(3).strip()),
                        quantidade=int(m_prod2.group(4).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Detectar próximo cliente ou fim de bloco
                if j > i + 2 and re.search(r"Cliente:\s*\d+", l):
                    break

                j += 1

            # Calcular total de volumes se não foi calculado
            if nota.total_volumes == 0:
                nota.total_volumes = sum(p.volumes for p in nota.produtos)

            carga.notas.append(nota)
            i = j
            continue

        i += 1

    return carga


def parse_carga_pdf_robust(pdf_path: str) -> Carga:
    """
    Parse robusto que também tenta extrair volumes das linhas de produto
    usando padrões mais flexíveis.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair dados do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)
        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()
        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Dividir texto em blocos por "Cliente:"
    texto_completo = "\n".join(linhas)
    blocos = re.split(r"(?=Cliente:\s*\d+)", texto_completo)

    for bloco in blocos:
        if not bloco.strip():
            continue

        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", bloco)
        if not m_cliente:
            continue

        nota = NotaFiscal(
            numero="",
            cliente_codigo=m_cliente.group(1).strip(),
            cliente_nome=m_cliente.group(2).strip(),
            cliente_fantasia="",
            cnpj="",
            endereco="",
            bairro="",
            cep="",
            telefone="",
            data_entrega="",
        )

        # CNPJ
        m = re.search(r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", bloco)
        if m:
            nota.cnpj = m.group(1).strip()
            nota.cliente_fantasia = m.group(2).strip()

        # Endereço
        m = re.search(
            r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
            bloco,
        )
        if m:
            nota.endereco = m.group(1).strip()
            nota.bairro = m.group(3).strip()

        # Data
        m = re.search(r"Data da entrega:\s*(\S+)", bloco)
        if m:
            nota.data_entrega = m.group(1).strip()
            if not carga.data:
                carga.data = nota.data_entrega

        # Telefone
        m = re.search(r"Telefone:\s*(.+?)\s+CEP:", bloco)
        if m:
            nota.telefone = m.group(1).strip()

        # CEP
        m = re.search(r"CEP:\s*(\d+)", bloco)
        if m:
            nota.cep = m.group(1).strip()

        # Nota
        m = re.search(r"NOTA:\s*(\d+)", bloco)
        if m:
            nota.numero = m.group(1).strip()

        # Produtos - padrão flexível
        for m_prod in re.finditer(
            r"(\d{4,6})\s+(.+?)\s{2,}(\d+)(?:\s+(\d+))?\s*$",
            bloco,
            re.MULTILINE,
        ):
            codigo = m_prod.group(1).strip()
            descricao = m_prod.group(2).strip()
            volumes = int(m_prod.group(3).strip())
            quantidade = (
                int(m_prod.group(4).strip()) if m_prod.group(4) else None
            )

            # Ignorar linhas que parecem ser CNPJ ou outros números
            if len(codigo) >= 4 and "CNPJ" not in descricao:
                prod = Produto(
                    codigo=codigo,
                    descricao=descricao,
                    volumes=volumes,
                    quantidade=quantidade,
                )
                nota.produtos.append(prod)

        nota.total_volumes = sum(p.volumes for p in nota.produtos)
        carga.notas.append(nota)

    return carga


def salvar_carga_json(carga: Carga, output_path: str):
    """Salva os dados da carga em formato JSON."""
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(carga.to_dict(), f, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Uso: python pdf_parser.py <caminho_do_pdf>")
        sys.exit(1)

    pdf_path = sys.argv[1]
    carga = parse_carga_pdf_robust(pdf_path)

    print(f"\n{'='*60}")
    print(f"CARREGAMENTO: {carga.numero_carregamento}")
    print(f"MOTORISTA: {carga.motorista_nome} ({carga.motorista_codigo})")
    print(f"DATA: {carga.data}")
    print(f"TOTAL DE NOTAS: {len(carga.notas)}")
    print(f"{'='*60}\n")

    for nota in carga.notas:
        print(f"NF-e {nota.numero} | {nota.cliente_nome}")
        print(f"  Fantasia: {nota.cliente_fantasia}")
        print(f"  CNPJ: {nota.cnpj}")
        print(f"  Endereço: {nota.endereco} - {nota.bairro}")
        print(f"  Volumes: {nota.total_volumes}")
        print(f"  Produtos: {len(nota.produtos)}")
        for p in nota.produtos:
            print(f"    - {p.codigo} {p.descricao} (Vol: {p.volumes})")
        print()

    # Salvar JSON
    output = pdf_path.replace(".pdf", "_carga.json")
    salvar_carga_json(carga, output)
    print(f"Dados salvos em: {output}")
  
Módulo de extração de dados de PDFs de carga.
Lê PDFs no formato Speers Foods e extrai todas as notas fiscais,
clientes, volumes, valores e produtos.
"""

import re
import subprocess
import json
import os
from dataclasses import dataclass, field, asdict
from typing import List, Optional


@dataclass
class Produto:
    codigo: str
    descricao: str
    volumes: int
    quantidade: Optional[int] = None


@dataclass
class NotaFiscal:
    numero: str
    cliente_codigo: str
    cliente_nome: str
    cliente_fantasia: str
    cnpj: str
    endereco: str
    bairro: str
    cep: str
    telefone: str
    data_entrega: str
    produtos: List[Produto] = field(default_factory=list)
    total_volumes: int = 0


@dataclass
class Carga:
    numero_carregamento: str
    motorista_codigo: str
    motorista_nome: str
    data: str
    notas: List[NotaFiscal] = field(default_factory=list)

    def to_dict(self):
        return {
            "numero_carregamento": self.numero_carregamento,
            "motorista_codigo": self.motorista_codigo,
            "motorista_nome": self.motorista_nome,
            "data": self.data,
            "notas": [
                {
                    "numero": n.numero,
                    "cliente_codigo": n.cliente_codigo,
                    "cliente_nome": n.cliente_nome,
                    "cliente_fantasia": n.cliente_fantasia,
                    "cnpj": n.cnpj,
                    "endereco": n.endereco,
                    "bairro": n.bairro,
                    "cep": n.cep,
                    "telefone": n.telefone,
                    "data_entrega": n.data_entrega,
                    "total_volumes": n.total_volumes,
                    "produtos": [
                        {
                            "codigo": p.codigo,
                            "descricao": p.descricao,
                            "volumes": p.volumes,
                            "quantidade": p.quantidade,
                        }
                        for p in n.produtos
                    ],
                }
                for n in self.notas
            ],
        }


def extrair_texto_pdf(pdf_path: str) -> str:
    """Extrai texto do PDF usando pdftotext."""
    try:
        result = subprocess.run(
            ["pdftotext", "-layout", pdf_path, "-"],
            capture_output=True,
            text=True,
            timeout=30,
        )
        return result.stdout
    except Exception as e:
        print(f"Erro ao extrair texto do PDF: {e}")
        return ""


def parse_carga_pdf(pdf_path: str) -> Carga:
    """
    Faz o parse completo de um PDF de carga no formato Speers Foods.
    Retorna um objeto Carga com todas as notas fiscais e produtos.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair número do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)

        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()

        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Encontrar blocos de clientes
    i = 0
    while i < len(linhas):
        linha = linhas[i]

        # Detectar início de bloco de cliente
        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", linha)
        if m_cliente:
            nota = NotaFiscal(
                numero="",
                cliente_codigo=m_cliente.group(1).strip(),
                cliente_nome=m_cliente.group(2).strip(),
                cliente_fantasia="",
                cnpj="",
                endereco="",
                bairro="",
                cep="",
                telefone="",
                data_entrega="",
            )

            # Procurar dados do cliente nas próximas linhas
            j = i + 1
            while j < min(i + 15, len(linhas)):
                l = linhas[j]

                # CNPJ e Fantasia
                m_cnpj = re.search(
                    r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", l
                )
                if m_cnpj:
                    nota.cnpj = m_cnpj.group(1).strip()
                    nota.cliente_fantasia = m_cnpj.group(2).strip()

                # Endereço
                m_end = re.search(
                    r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
                    l,
                )
                if m_end:
                    nota.endereco = m_end.group(1).strip()
                    nota.bairro = m_end.group(3).strip()

                # Data de entrega, telefone, CEP
                m_data = re.search(r"Data da entrega:\s*(\S+)", l)
                if m_data:
                    nota.data_entrega = m_data.group(1).strip()
                    if not carga.data:
                        carga.data = nota.data_entrega

                m_tel = re.search(r"Telefone:\s*(\S+)", l)
                if m_tel:
                    nota.telefone = m_tel.group(1).strip()

                m_cep = re.search(r"CEP:\s*(\d+)", l)
                if m_cep:
                    nota.cep = m_cep.group(1).strip()

                # Nota fiscal
                m_nota = re.search(r"NOTA:\s*(\d+)", l)
                if m_nota:
                    nota.numero = m_nota.group(1).strip()

                # Produtos (linhas com código numérico no início)
                m_prod = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s*$", l
                )
                if m_prod:
                    prod = Produto(
                        codigo=m_prod.group(1).strip(),
                        descricao=m_prod.group(2).strip(),
                        volumes=int(m_prod.group(3).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Produto com quantidade entregue
                m_prod2 = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s+(\d+)\s*$", l
                )
                if m_prod2 and not m_prod:
                    prod = Produto(
                        codigo=m_prod2.group(1).strip(),
                        descricao=m_prod2.group(2).strip(),
                        volumes=int(m_prod2.group(3).strip()),
                        quantidade=int(m_prod2.group(4).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Detectar próximo cliente ou fim de bloco
                if j > i + 2 and re.search(r"Cliente:\s*\d+", l):
                    break

                j += 1

            # Calcular total de volumes se não foi calculado
            if nota.total_volumes == 0:
                nota.total_volumes = sum(p.volumes for p in nota.produtos)

            carga.notas.append(nota)
            i = j
            continue

        i += 1

    return carga


def parse_carga_pdf_robust(pdf_path: str) -> Carga:
    """
    Parse robusto que também tenta extrair volumes das linhas de produto
    usando padrões mais flexíveis.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair dados do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)
        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()
        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Dividir texto em blocos por "Cliente:"
    texto_completo = "\n".join(linhas)
    blocos = re.split(r"(?=Cliente:\s*\d+)", texto_completo)

    for bloco in blocos:
        if not bloco.strip():
            continue

        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", bloco)
        if not m_cliente:
            continue

        nota = NotaFiscal(
            numero="",
            cliente_codigo=m_cliente.group(1).strip(),
            cliente_nome=m_cliente.group(2).strip(),
            cliente_fantasia="",
            cnpj="",
            endereco="",
            bairro="",
            cep="",
            telefone="",
            data_entrega="",
        )

        # CNPJ
        m = re.search(r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", bloco)
        if m:
            nota.cnpj = m.group(1).strip()
            nota.cliente_fantasia = m.group(2).strip()

        # Endereço
        m = re.search(
            r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
            bloco,
        )
        if m:
            nota.endereco = m.group(1).strip()
            nota.bairro = m.group(3).strip()

        # Data
        m = re.search(r"Data da entrega:\s*(\S+)", bloco)
        if m:
            nota.data_entrega = m.group(1).strip()
            if not carga.data:
                carga.data = nota.data_entrega

        # Telefone
        m = re.search(r"Telefone:\s*(.+?)\s+CEP:", bloco)
        if m:
            nota.telefone = m.group(1).strip()

        # CEP
        m = re.search(r"CEP:\s*(\d+)", bloco)
        if m:
            nota.cep = m.group(1).strip()

        # Nota
        m = re.search(r"NOTA:\s*(\d+)", bloco)
        if m:
            nota.numero = m.group(1).strip()

        # Produtos - padrão flexível
        for m_prod in re.finditer(
            r"(\d{4,6})\s+(.+?)\s{2,}(\d+)(?:\s+(\d+))?\s*$",
            bloco,
            re.MULTILINE,
        ):
            codigo = m_prod.group(1).strip()
            descricao = m_prod.group(2).strip()
            volumes = int(m_prod.group(3).strip())
            quantidade = (
                int(m_prod.group(4).strip()) if m_prod.group(4) else None
            )

            # Ignorar linhas que parecem ser CNPJ ou outros números
            if len(codigo) >= 4 and "CNPJ" not in descricao:
                prod = Produto(
                    codigo=codigo,
                    descricao=descricao,
                    volumes=volumes,
                    quantidade=quantidade,
                )
                nota.produtos.append(prod)

        nota.total_volumes = sum(p.volumes for p in nota.produtos)
        carga.notas.append(nota)

    return carga


def salvar_carga_json(carga: Carga, output_path: str):
    """Salva os dados da carga em formato JSON."""
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(carga.to_dict(), f, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Uso: python pdf_parser.py <caminho_do_pdf>")
        sys.exit(1)

    pdf_path = sys.argv[1]
    carga = parse_carga_pdf_robust(pdf_path)

    print(f"\n{'='*60}")
    print(f"CARREGAMENTO: {carga.numero_carregamento}")
    print(f"MOTORISTA: {carga.motorista_nome} ({carga.motorista_codigo})")
    print(f"DATA: {carga.data}")
    print(f"TOTAL DE NOTAS: {len(carga.notas)}")
    print(f"{'='*60}\n")

    for nota in carga.notas:
        print(f"NF-e {nota.numero} | {nota.cliente_nome}")
        print(f"  Fantasia: {nota.cliente_fantasia}")
        print(f"  CNPJ: {nota.cnpj}")
        print(f"  Endereço: {nota.endereco} - {nota.bairro}")
        print(f"  Volumes: {nota.total_volumes}")
        print(f"  Produtos: {len(nota.produtos)}")
        for p in nota.produtos:
            print(f"    - {p.codigo} {p.descricao} (Vol: {p.volumes})")
        print()

    # Salvar JSON
    output = pdf_path.replace(".pdf", "_carga.json")
    salvar_carga_json(carga, output)
    print(f"Dados salvos em: {output}")
           pendentes_agora = len([c for c in canhoteiras_dict if c.get("status") == "PENDENTE"])

        msg = f"""📊 *ANÁLISE DA CANHOTEIRA*
━━━━━━━━━━━━━━━━━━━━
📸 Canhoteiras analisadas nesta foto: *{analisadas}*
✅ Assinadas: *{entregues_agora}*
❌ Sem assinatura: *{pendentes_agora}*

📦 *STATUS GERAL DA CARGA {carga['numero_carregamento']}:*
✅ Entregues: *{len(entregues)}/{len(entregas)}*
❌ Pendentes: *{len(pendentes)}/{len(entregas)}*
"""

        enviar_mensagem_grupo(grupo_id, msg)

        # Enviar cobrança para cada nota pendente encontrada
        for c_data in canhoteiras_dict:
            if c_data.get("status") == "PENDENTE":
                nf = c_data.get("nf_numero", "")
                # Buscar dados completos da entrega
                entrega_db = None
                for e in entregas:
                    nf_clean = "".join(c for c in str(e["nf_numero"]) if c.isdigit())
                    if nf_clean == "".join(c for c in nf if c.isdigit()):
                        entrega_db = e
                        break

                if entrega_db:
                    resultado = ResultadoEntrega(
                        nf_numero=entrega_db["nf_numero"],
                        cliente_codigo=entrega_db.get("cliente_codigo", ""),
                        cliente_nome=entrega_db.get("cliente_nome", ""),
                        cliente_fantasia=entrega_db.get("cliente_fantasia", ""),
                        cnpj=entrega_db.get("cnpj", ""),
                        endereco=entrega_db.get("endereco", ""),
                        telefone=entrega_db.get("telefone", ""),
                        total_volumes=entrega_db.get("total_volumes", 0),
                        produtos=[],
                        status="PENDENTE",
                    )
                    cobranca = gerar_mensagem_cobranca(resultado, carga["numero_carregamento"])
                    enviar_mensagem_grupo(grupo_id, cobranca)

                    registrar_mensagem(
                        carga_id, nf, "COBRANCA_MOTIVO", cobranca, "BOT", grupo_id
                    )

    except Exception as e:
        print(f"Erro ao processar imagem: {e}")
        traceback.print_exc()
        enviar_mensagem_grupo(
            grupo_id,
            f"❌ Erro ao analisar a canhoteira: {str(e)}\nTente enviar a foto novamente com melhor qualidade."
        )


async def processar_resposta_motivo(data: dict, grupo_id: str, texto: str):
    """Processa resposta do motorista com motivo de devolução."""
    sender = get_sender(data)

    carga = get_carga_ativa_por_grupo(grupo_id)
    if not carga:
        return

    carga_id = carga["id"]

    # Verificar se há entregas aguardando motivo
    aguardando = get_entregas_aguardando_motivo(carga_id)
    if not aguardando:
        return

    # Interpretar o motivo
    motivo = interpretar_motivo(texto)

    # Registrar motivo para a primeira entrega aguardando
    entrega = aguardando[0]
    registrar_motivo_devolucao(carga_id, entrega["nf_numero"], motivo)

    # Enviar confirmação
    confirmacao = gerar_confirmacao_motivo(entrega["nf_numero"], motivo)
    enviar_mensagem_grupo(grupo_id, confirmacao)

    registrar_mensagem(
        carga_id, entrega["nf_numero"], "MOTIVO_REGISTRADO", motivo, sender, grupo_id
    )

    # Verificar se ainda há entregas aguardando
    ainda_aguardando = get_entregas_aguardando_motivo(carga_id)
    if ainda_aguardando:
        prox = ainda_aguardando[0]
        nome = prox.get("cliente_fantasia") or prox.get("cliente_nome", "")
        enviar_mensagem_grupo(
            grupo_id,
            f"⚠️ Ainda há *{len(ainda_aguardando)}* nota(s) pendente(s) de motivo.\n"
            f"Próxima: NF-e *{prox['nf_numero']}* — {nome}\n\n"
            f"_Informe o motivo (1-8 ou texto livre):_"
        )
    else:
        # Todas as entregas com motivo registrado
        atualizar_status_carga(carga_id)
        carga_atualizada = get_carga(carga_id)

        enviar_mensagem_grupo(
            grupo_id,
            f"✅ *Todos os motivos foram registrados!*\n\n"
            f"📊 *RESUMO FINAL — Carga {carga['numero_carregamento']}:*\n"
            f"✅ Entregues: *{carga_atualizada['total_entregues']}*\n"
            f"❌ Devoluções: *{carga_atualizada['total_pendentes']}*\n\n"
            f"_Carga finalizada. Dados salvos no sistema._"
        )
        finalizar_carga(carga_id)


async def processar_comando(data: dict, grupo_id: str, texto: str):
    """Processa comandos especiais do administrador."""
    sender = get_sender(data)
    texto_lower = texto.lower().strip()

    if texto_lower in ("/status", "!status", "status"):
        carga = get_carga_ativa_por_grupo(grupo_id)
        if not carga:
            enviar_mensagem_grupo(grupo_id, "ℹ️ Nenhuma carga ativa neste grupo.")
            return True

        entregas = get_entregas_por_carga(carga["id"])
        entregues = [e for e in entregas if e["status"] == "ENTREGUE"]
        pendentes = [e for e in entregas if e["status"] != "ENTREGUE"]

        msg = f"""📊 *STATUS DA CARGA {carga['numero_carregamento']}*
━━━━━━━━━━━━━━━━━━━━
👤 Motorista: *{carga['motorista_nome']}*
📅 Data: *{carga['data_carga']}*

✅ Entregues: *{len(entregues)}/{len(entregas)}*
❌ Pendentes: *{len(pendentes)}/{len(entregas)}*
"""
        if pendentes:
            msg += "\n📋 *Notas pendentes:*\n"
            for p in pendentes:
                nome = p.get("cliente_fantasia") or p.get("cliente_nome", "")
                motivo = p.get("motivo_devolucao", "")
                status_motivo = f" — Motivo: {motivo}" if motivo else " — ⚠️ Sem motivo"
                msg += f"   ❌ NF-e *{p['nf_numero']}* — {nome}{status_motivo}\n"

        enviar_mensagem_grupo(grupo_id, msg)
        return True

    if texto_lower in ("/finalizar", "!finalizar", "finalizar carga"):
        if not is_admin(sender):
            enviar_mensagem_grupo(grupo_id, "⚠️ Apenas administradores podem finalizar cargas.")
            return True

        carga = get_carga_ativa_por_grupo(grupo_id)
        if not carga:
            enviar_mensagem_grupo(grupo_id, "ℹ️ Nenhuma carga ativa para finalizar.")
            return True

        finalizar_carga(carga["id"])
        enviar_mensagem_grupo(
            grupo_id,
            f"✅ Carga *{carga['numero_carregamento']}* finalizada com sucesso!\n"
            f"Envie um novo PDF para iniciar outra carga."
        )
        return True

    if texto_lower in ("/ajuda", "!ajuda", "/help", "ajuda"):
        msg = """📖 *COMANDOS DISPONÍVEIS*
━━━━━━━━━━━━━━━━━━━━

📄 *Enviar PDF* → Registra nova carga
📸 *Enviar foto* → Analisa canhoteiras
1️⃣-8️⃣ *Número* → Informa motivo de devolução

📊 *status* → Ver status da carga atual
🏁 *finalizar* → Finalizar carga (ADM)
❓ *ajuda* → Ver esta mensagem

━━━━━━━━━━━━━━━━━━━━
_Sistema de Controle de Entregas_"""
        enviar_mensagem_grupo(grupo_id, msg)
        return True

    return False


# ==================== WEBHOOK ====================

@app.post("/webhook")
async def webhook(request: Request):
    """Endpoint de webhook para receber eventos da Evolution API."""
    try:
        data = await request.json()
    except Exception:
        return JSONResponse({"status": "error", "message": "Invalid JSON"}, status_code=400)

    event = data.get("event", "")

    # Processar apenas mensagens recebidas
    if event != "messages.upsert":
        return JSONResponse({"status": "ok", "event": event})

    # Ignorar mensagens enviadas pelo próprio bot
    key = data.get("data", {}).get("key", {})
    if key.get("fromMe", False):
        return JSONResponse({"status": "ok", "message": "self message"})

    # Verificar se é mensagem de grupo
    if not is_group_message(data):
        return JSONResponse({"status": "ok", "message": "not group"})

    grupo_id = get_group_id(data)
    msg_type = get_message_type(data)
    texto = get_message_text(data)

    print(f"[{datetime.now()}] Mensagem recebida: tipo={msg_type}, grupo={grupo_id}, texto={texto[:50]}")

    try:
        # Processar comandos de texto
        if msg_type == "text":
            # Verificar se é comando
            is_command = await processar_comando(data, grupo_id, texto)
            if not is_command:
                # Verificar se é resposta de motivo
                await processar_resposta_motivo(data, grupo_id, texto)

        # Processar PDF de carga
        elif msg_type == "pdf":
            await processar_pdf_carga(data, grupo_id)

        # Processar imagem de canhoteira
        elif msg_type == "image":
            await processar_imagem_canhoteira(data, grupo_id)

    except Exception as e:
        print(f"Erro ao processar mensagem: {e}")
        traceback.print_exc()

    return JSONResponse({"status": "ok"})


@app.get("/health")
async def health():
    """Health check endpoint."""
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}


@app.get("/")
async def root():
    """Root endpoint."""
    return {
        "name": "Controle de Entregas - Bot WhatsApp",
        "version": "1.0.0",
        "status": "running",
    }
import sqlite3
import os
from datetime import datetime

DB_PATH = os.environ.get("DB_PATH", "data/entregas.db")

def get_connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = get_connection()
    
    # Tabela de motoristas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS motoristas (
            codigo TEXT PRIMARY KEY,
            nome TEXT NOT NULL
        )
    """)
    
    # Tabela de cargas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS cargas (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            numero_carregamento TEXT NOT NULL,
            motorista_codigo TEXT NOT NULL,
            motorista_nome TEXT,
            data_carga TEXT,
            grupo_whatsapp TEXT,
            status TEXT DEFAULT 'ATIVA',
            FOREIGN KEY (motorista_codigo) REFERENCES motoristas (codigo)
        )
    """)
    
    # Tabela de entregas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS entregas (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            carga_id INTEGER NOT NULL,
            nf_numero TEXT NOT NULL,
            cliente_codigo TEXT,
            cliente_nome TEXT,
            cliente_fantasia TEXT,
            cnpj TEXT,
            endereco TEXT,
            telefone TEXT,
            total_volumes INTEGER,
            status TEXT DEFAULT 'PENDENTE',
            motivo_devolucao TEXT,
            data_recebimento TEXT,
            recebedor_nome TEXT,
            recebedor_documento TEXT,
            temperatura TEXT,
            FOREIGN KEY (carga_id) REFERENCES cargas (id)
        )
    """)
    
    # Tabela de logs de mensagens
    conn.execute("""
        CREATE TABLE IF NOT EXISTS mensagens (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            carga_id INTEGER,
            nf_numero TEXT,
            tipo TEXT,
            conteudo TEXT,
            remetente TEXT,
            grupo_id TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    conn.commit()
    conn.close()

def registrar_motorista(codigo, nome):
    conn = get_connection()
    conn.execute(
        "INSERT OR REPLACE INTO motoristas (codigo, nome) VALUES (?, ?)",
        (codigo, nome)
    )
    conn.commit()
    conn.close()

def listar_motoristas():
    conn = get_connection()
    rows = conn.execute("SELECT * FROM motoristas").fetchall()
    conn.close()
    return [dict(r) for r in rows]

def criar_carga(numero, motorista_cod, motorista_nome, data, grupo_id):
    registrar_motorista(motorista_cod, motorista_nome)
    conn = get_connection()
    cursor = conn.execute(
        "INSERT INTO cargas (numero_carregamento, motorista_codigo, motorista_nome, data_carga, grupo_whatsapp) VALUES (?, ?, ?, ?, ?)",
        (numero, motorista_cod, motorista_nome, data, grupo_id)
    )
    carga_id = cursor.lastrowid
    conn.commit()
    conn.close()
    return carga_id

def registrar_entrega(carga_id, nf_data):
    conn = get_connection()
    conn.execute("""
        INSERT INTO entregas (
            carga_id, nf_numero, cliente_codigo, cliente_nome, cliente_fantasia, 
            cnpj, endereco, telefone, total_volumes, status
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        carga_id, nf_data['numero'], nf_data.get('cliente_codigo'), nf_data.get('cliente_nome'),
        nf_data.get('cliente_fantasia'), nf_data.get('cnpj'), nf_data.get('endereco'),
        nf_data.get('telefone'), nf_data.get('total_volumes'), 'PENDENTE'
    ))
    conn.commit()
    conn.close()

def get_carga_ativa_por_grupo(grupo_id):
    conn = get_connection()
    row = conn.execute(
        "SELECT * FROM cargas WHERE grupo_whatsapp = ? AND status = 'ATIVA' ORDER BY id DESC LIMIT 1",
        (grupo_id,)
    ).fetchone()
    conn.close()
    return dict(row) if row else None

def get_entregas_por_carga(carga_id):
    conn = get_connection()
    rows = conn.execute("SELECT * FROM entregas WHERE carga_id = ?", (carga_id,)).fetchall()
    conn.close()
    return [dict(r) for r in rows]

def registrar_motivo_devolucao(carga_id, nf_numero, motivo):
    conn = get_connection()
    conn.execute(
        "UPDATE entregas SET motivo_devolucao = ?, status = 'DEVOLVIDA' WHERE carga_id = ? AND nf_numero = ?",
        (motivo, carga_id, nf_numero)
    )
    conn.commit()
    conn.close()

def registrar_entrega_sucesso(carga_id, nf_numero, dados):
    conn = get_connection()
    conn.execute("""
        UPDATE entregas SET 
            status = 'ENTREGUE',
            data_recebimento = ?,
            recebedor_nome = ?,
            recebedor_documento = ?,
            temperatura = ?
        WHERE carga_id = ? AND nf_numero = ?
    """, (
        dados.get('data_recebimento'), dados.get('recebedor_nome'),
        dados.get('recebedor_documento'), dados.get('temperatura'),
        carga_id, nf_numero
    ))
    conn.commit()
    conn.close()

def finalizar_carga(carga_id):
    conn = get_connection()
    conn.execute("UPDATE cargas SET status = 'FINALIZADA' WHERE id = ?", (carga_id,))
    conn.commit()
    conn.close()

def registrar_mensagem(carga_id, nf_numero, tipo, conteudo, remetente, grupo_id):
    conn = get_connection()
    conn.execute("""
        INSERT INTO mensagens (carga_id, nf_numero, tipo, conteudo, remetente, grupo_id)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (carga_id, nf_numero, tipo, conteudo, remetente, grupo_id))
    conn.commit()
    conn.close()

def get_entregas_aguardando_motivo(carga_id):
    conn = get_connection()
    rows = conn.execute(
        "SELECT * FROM entregas WHERE carga_id = ? AND status = 'PENDENTE' AND (motivo_devolucao IS NULL OR motivo_devolucao = '')",
        (carga_id,)
    ).fetchall()
    conn.close()
    return [dict(r) for r in rows]

# Funções para relatórios (DOC-20260302-WA0026)
def get_ranking_motoristas(data_inicio, data_fim):
    conn = get_connection()
    query = """
        SELECT m.nome, COUNT(e.id) as total, SUM(CASE WHEN e.status='ENTREGUE' THEN 1 ELSE 0 END) as entregues
        FROM motoristas m
        JOIN cargas c ON c.motorista_codigo = m.codigo
        JOIN entregas e ON e.carga_id = c.id
        WHERE c.data_carga BETWEEN ? AND ?
        GROUP BY m.nome
    """
    rows = conn.execute(query, (data_inicio, data_fim)).fetchall()
    conn.close()
    return [dict(r) for r in rows]

# Inicializar banco ao importar
os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
init_db()
"""
Módulo de extração de dados de PDFs de carga.
Lê PDFs no formato Speers Foods e extrai todas as notas fiscais,
clientes, volumes, valores e produtos.
"""

import re
import subprocess
import json
import os
from dataclasses import dataclass, field, asdict
from typing import List, Optional


@dataclass
class Produto:
    codigo: str
    descricao: str
    volumes: int
    quantidade: Optional[int] = None


@dataclass
class NotaFiscal:
    numero: str
    cliente_codigo: str
    cliente_nome: str
    cliente_fantasia: str
    cnpj: str
    endereco: str
    bairro: str
    cep: str
    telefone: str
    data_entrega: str
    produtos: List[Produto] = field(default_factory=list)
    total_volumes: int = 0
    observacao: str = ""


@dataclass
class Carga:
    numero_carregamento: str
    motorista_codigo: str
    motorista_nome: str
    data: str
    notas: List[NotaFiscal] = field(default_factory=list)

    def to_dict(self):
        return {
            "numero_carregamento": self.numero_carregamento,
            "motorista_codigo": self.motorista_codigo,
            "motorista_nome": self.motorista_nome,
            "data": self.data,
            "notas": [
                {
                    "numero": n.numero,
                    "cliente_codigo": n.cliente_codigo,
                    "cliente_nome": n.cliente_nome,
                    "cliente_fantasia": n.cliente_fantasia,
                    "cnpj": n.cnpj,
                    "endereco": n.endereco,
                    "bairro": n.bairro,
                    "cep": n.cep,
                    "telefone": n.telefone,
                    "data_entrega": n.data_entrega,
                    "total_volumes": n.total_volumes,
                    "observacao": n.observacao,
                    "produtos": [
                        {
                            "codigo": p.codigo,
                            "descricao": p.descricao,
                            "volumes": p.volumes,
                            "quantidade": p.quantidade,
                        }
                        for p in n.produtos
                    ],
                }
                for n in self.notas
            ],
        }


def extrair_texto_pdf(pdf_path: str) -> str:
    """Extrai texto do PDF usando pdftotext."""
    try:
        result = subprocess.run(
            ["pdftotext", "-layout", pdf_path, "-"],
            capture_output=True,
            text=True,
            timeout=30,
        )
        return result.stdout
    except Exception as e:
        print(f"Erro ao extrair texto do PDF: {e}")
        return ""


def parse_carga_pdf(pdf_path: str) -> Carga:
    """
    Faz o parse completo de um PDF de carga no formato Speers Foods.
    Retorna um objeto Carga com todas as notas fiscais e produtos.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair número do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)

        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()

        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Encontrar blocos de clientes
    i = 0
    while i < len(linhas):
        linha = linhas[i]

        # Detectar início de bloco de cliente
        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", linha)
        if m_cliente:
            nota = NotaFiscal(
                numero="",
                cliente_codigo=m_cliente.group(1).strip(),
                cliente_nome=m_cliente.group(2).strip(),
                cliente_fantasia="",
                cnpj="",
                endereco="",
                bairro="",
                cep="",
                telefone="",
                data_entrega="",
            )

            # Procurar dados do cliente nas próximas linhas
            j = i + 1
            while j < min(i + 15, len(linhas)):
                l = linhas[j]

                # CNPJ e Fantasia
                m_cnpj = re.search(
                    r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", l
                )
                if m_cnpj:
                    nota.cnpj = m_cnpj.group(1).strip()
                    nota.cliente_fantasia = m_cnpj.group(2).strip()

                # Endereço
                m_end = re.search(
                    r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
                    l,
                )
                if m_end:
                    nota.endereco = m_end.group(1).strip()
                    nota.bairro = m_end.group(3).strip()

                # Data de entrega, telefone, CEP
                m_data = re.search(r"Data da entrega:\s*(\S+)", l)
                if m_data:
                    nota.data_entrega = m_data.group(1).strip()
                    if not carga.data:
                        carga.data = nota.data_entrega

                m_tel = re.search(r"Telefone:\s*(\S+)", l)
                if m_tel:
                    nota.telefone = m_tel.group(1).strip()

                m_cep = re.search(r"CEP:\s*(\d+)", l)
                if m_cep:
                    nota.cep = m_cep.group(1).strip()

                # Nota fiscal
                m_nota = re.search(r"NOTA:\s*(\d+)", l)
                if m_nota:
                    nota.numero = m_nota.group(1).strip()

                # Produtos (linhas com código numérico no início)
                m_prod = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s*$", l
                )
                if m_prod:
                    prod = Produto(
                        codigo=m_prod.group(1).strip(),
                        descricao=m_prod.group(2).strip(),
                        volumes=int(m_prod.group(3).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Produto com quantidade entregue
                m_prod2 = re.search(
                    r"^\s*(\d{4,6})\s+(.+?)\s+(\d+)\s+(\d+)\s*$", l
                )
                if m_prod2 and not m_prod:
                    prod = Produto(
                        codigo=m_prod2.group(1).strip(),
                        descricao=m_prod2.group(2).strip(),
                        volumes=int(m_prod2.group(3).strip()),
                        quantidade=int(m_prod2.group(4).strip()),
                    )
                    nota.produtos.append(prod)
                    nota.total_volumes += prod.volumes

                # Detectar próximo cliente ou fim de bloco
                if j > i + 2 and re.search(r"Cliente:\s*\d+", l):
                    break

                j += 1

            # Calcular total de volumes se não foi calculado
            if nota.total_volumes == 0:
                nota.total_volumes = sum(p.volumes for p in nota.produtos)

            carga.notas.append(nota)
            i = j
            continue

        i += 1

    return carga


def parse_carga_pdf_robust(pdf_path: str) -> Carga:
    """
    Parse robusto que também tenta extrair volumes das linhas de produto
    usando padrões mais flexíveis.
    """
    texto = extrair_texto_pdf(pdf_path)
    if not texto:
        raise ValueError(f"Não foi possível extrair texto do PDF: {pdf_path}")

    linhas = texto.split("\n")

    # Extrair dados do carregamento
    carregamento = ""
    motorista_cod = ""
    motorista_nome = ""

    for linha in linhas:
        m = re.search(r"CARREGAMENTO:\s*(\d+)", linha)
        if m:
            carregamento = m.group(1)
        m = re.search(r"MOTORISTA\s+(\d+)\s+(.+)", linha)
        if m:
            motorista_cod = m.group(1).strip()
            motorista_nome = m.group(2).strip()
        if carregamento and motorista_cod:
            break

    carga = Carga(
        numero_carregamento=carregamento,
        motorista_codigo=motorista_cod,
        motorista_nome=motorista_nome,
        data="",
    )

    # Dividir texto em blocos por "Cliente:"
    texto_completo = "\n".join(linhas)
    blocos = re.split(r"(?=Cliente:\s*\d+)", texto_completo)

    for bloco in blocos:
        if not bloco.strip():
            continue

        m_cliente = re.search(r"Cliente:\s*(\d+)\s+(.+)", bloco)
        if not m_cliente:
            continue

        nota = NotaFiscal(
            numero="",
            cliente_codigo=m_cliente.group(1).strip(),
            cliente_nome=m_cliente.group(2).strip(),
            cliente_fantasia="",
            cnpj="",
            endereco="",
            bairro="",
            cep="",
            telefone="",
            data_entrega="",
        )

        # CNPJ
        m = re.search(r"C\.N\.P\.J\.:\s*([\d./-]+)\s+FANTASIA\s*(.*)", bloco)
        if m:
            nota.cnpj = m.group(1).strip()
            nota.cliente_fantasia = m.group(2).strip()

        # Endereço
        m = re.search(
            r"Endere[çc]o:\s*(.+?)(?:\s+N[º°]:\s*(\S+))?\s+Bairro:\s*(.+)",
            bloco,
        )
        if m:
            nota.endereco = m.group(1).strip()
            nota.bairro = m.group(3).strip()

        # Data
        m = re.search(r"Data da entrega:\s*(\S+)", bloco)
        if m:
            nota.data_entrega = m.group(1).strip()
            if not carga.data:
                carga.data = nota.data_entrega

        # Telefone
        m = re.search(r"Telefone:\s*(.+?)\s+CEP:", bloco)
        if m:
            nota.telefone = m.group(1).strip()

        # CEP
        m = re.search(r"CEP:\s*(\d+)", bloco)
        if m:
            nota.cep = m.group(1).strip()

        # Nota
        m = re.search(r"NOTA:\s*(\d+)", bloco)
        if m:
            nota.numero = m.group(1).strip()

        # Observação
        m_obs = re.search(r"OBSERVA[ÇC][ÕO]ES:\s*(.+?)(?=(?:Cliente:|\Z))", bloco, re.DOTALL)
        if m_obs:
            nota.observacao = m_obs.group(1).strip()

        # Produtos - padrão flexível
        for m_prod in re.finditer(
            r"(\d{4,6})\s+(.+?)\s{2,}(\d+)(?:\s+(\d+))?\s*$",
            bloco,
            re.MULTILINE,
        ):
            codigo = m_prod.group(1).strip()
            descricao = m_prod.group(2).strip()
            volumes = int(m_prod.group(3).strip())
            quantidade = (
                int(m_prod.group(4).strip()) if m_prod.group(4) else None
            )

            # Ignorar linhas que parecem ser CNPJ ou outros números
            if len(codigo) >= 4 and "CNPJ" not in descricao:
                prod = Produto(
                    codigo=codigo,
                    descricao=descricao,
                    volumes=volumes,
                    quantidade=quantidade,
                )
                nota.produtos.append(prod)

        nota.total_volumes = sum(p.volumes for p in nota.produtos)
        carga.notas.append(nota)

    return carga


def salvar_carga_json(carga: Carga, output_path: str):
    """Salva os dados da carga em formato JSON."""
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(carga.to_dict(), f, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Uso: python pdf_parser.py <caminho_do_pdf>")
        sys.exit(1)

    pdf_path = sys.argv[1]
    carga = parse_carga_pdf_robust(pdf_path)

    print(f"\n{'='*60}")
    print(f"CARREGAMENTO: {carga.numero_carregamento}")
    print(f"MOTORISTA: {carga.motorista_nome} ({carga.motorista_codigo})")
    print(f"DATA: {carga.data}")
    print(f"TOTAL DE NOTAS: {len(carga.notas)}")
    print(f"{'='*60}\n")

    for nota in carga.notas:
        print(f"NF-e {nota.numero} | {nota.cliente_nome}")
        print(f"  Fantasia: {nota.cliente_fantasia}")
        print(f"  CNPJ: {nota.cnpj}")
        print(f"  Endereço: {nota.endereco} - {nota.bairro}")
        print(f"  Volumes: {nota.total_volumes}")
        print(f"  Produtos: {len(nota.produtos)}")
        if nota.observacao:
            print(f"  Observação: {nota.observacao}")
        for p in nota.produtos:
            print(f"    - {p.codigo} {p.descricao} (Vol: {p.volumes})")
        print()

    # Salvar JSON
    output = pdf_path.replace(".pdf", "_carga.json")
    salvar_carga_json(carga, output)
    print(f"Dados salvos em: {output}")
import os
import json
import traceback
import requests
from datetime import datetime
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app.database import (
    criar_carga, registrar_entrega, get_carga_ativa_por_grupo,
    get_entregas_por_carga, registrar_motivo_devolucao,
    registrar_entrega_sucesso, finalizar_carga, registrar_mensagem,
    get_entregas_aguardando_motivo
)
from app.pdf_parser import parse_carga_pdf_robust
from app.image_analyzer import analisar_multiplas_imagens
from app.data_matcher import cruzar_dados, ResultadoEntrega, interpretar_motivo, gerar_mensagem_cobranca, gerar_confirmacao_motivo

app = FastAPI()

# Configurações da Evolution API
EVOLUTION_API_URL = os.environ.get("EVOLUTION_API_URL")
EVOLUTION_API_KEY = os.environ.get("EVOLUTION_API_KEY")
EVOLUTION_INSTANCE = os.environ.get("EVOLUTION_INSTANCE")
ADMIN_NUMBERS = os.environ.get("ADMIN_NUMBERS", "").split(",")

def is_admin(number):
    clean_number = "".join(c for c in number if c.isdigit())
    return any(clean_number.endswith(admin) for admin in ADMIN_NUMBERS)

def enviar_mensagem_grupo(grupo_id, texto):
    url = f"{EVOLUTION_API_URL}/message/sendText/{EVOLUTION_INSTANCE}"
    headers = {"apikey": EVOLUTION_API_KEY, "Content-Type": "application/json"}
    payload = {
        "number": grupo_id,
        "options": {"delay": 1200, "presence": "composing", "linkPreview": False},
        "textMessage": {"text": texto}
    }
    response = requests.post(url, headers=headers, json=payload)
    return response.json()

def baixar_arquivo_evolution(media_key, direct_path):
    # Simulação da função de download da Evolution API
    # Na prática, você usaria a URL fornecida no webhook
    pass

def get_group_id(data):
    return data.get("data", {}).get("key", {}).get("remoteJid")

def get_message_type(data):
    msg = data.get("data", {}).get("message", {})
    if "conversation" in msg or "extendedTextMessage" in msg:
        return "text"
    if "imageMessage" in msg:
        return "image"
    if "documentMessage" in msg:
        doc = msg["documentMessage"]
        if doc.get("mimetype") == "application/pdf":
            return "pdf"
    return "unknown"

def get_message_text(data):
    msg = data.get("data", {}).get("message", {})
    if "conversation" in msg:
        return msg["conversation"]
    if "extendedTextMessage" in msg:
        return msg["extendedTextMessage"].get("text", "")
    return ""

def get_sender(data):
    return data.get("data", {}).get("key", {}).get("participant") or data.get("data", {}).get("key", {}).get("remoteJid")

def is_group_message(data):
    return "@g.us" in get_group_id(data)

async def processar_pdf_carga(data, grupo_id):
    """Processa o PDF de carga enviado no grupo."""
    try:
        msg = data.get("data", {}).get("message", {}).get("documentMessage", {})
        # Aqui você baixaria o arquivo real usando a Evolution API
        # Para este exemplo, vamos simular que o arquivo foi salvo em /tmp/carga.pdf
        pdf_path = "/tmp/carga.pdf"
        
        # Simulação: No ambiente real, você usaria requests para baixar o binário
        enviar_mensagem_grupo(grupo_id, "⏳ Processando PDF de carga, aguarde...")
        
        carga = parse_carga_pdf_robust(pdf_path)
        carga_id = criar_carga(
            carga.numero_carregamento,
            carga.motorista_codigo,
            carga.motorista_nome,
            carga.data,
            grupo_id
        )
        
        alertas_obs = []
        for nota in carga.notas:
            registrar_entrega(carga_id, nota.__dict__)
            if nota.observacao:
                alertas_obs.append(f"⚠️ *NF-e {nota.numero}*: {nota.observacao}")
        
        resumo = f"""✅ *CARGA REGISTRADA COM SUCESSO!*
━━━━━━━━━━━━━━━━━━━━
🚛 Carregamento: *{carga.numero_carregamento}*
👤 Motorista: *{carga.motorista_nome}*
📅 Data: *{carga.data}*
📋 Total de Notas: *{len(carga.notas)}*
━━━━━━━━━━━━━━━━━━━━
"""
        if alertas_obs:
            resumo += "\n🔔 *ALERTAS DE OBSERVAÇÃO:*\n" + "\n".join(alertas_obs) + "\n━━━━━━━━━━━━━━━━━━━━\n"
            
        resumo += "\n_Envie as fotos das canhoteiras para iniciar a conferência._"
        
        enviar_mensagem_grupo(grupo_id, resumo)
        
    except Exception as e:
        print(f"Erro ao processar PDF: {e}")
        traceback.print_exc()
        enviar_mensagem_grupo(grupo_id, f"❌ Erro ao processar o PDF: {str(e)}")

async def processar_imagem_canhoteira(data, grupo_id):
    """Processa imagens de canhoteiras enviadas no grupo."""
    try:
        carga = get_carga_ativa_por_grupo(grupo_id)
        if not carga:
            return

        # Simulação de download e análise
        # No real, você coletaria múltiplas imagens se enviadas em lote
        enviar_mensagem_grupo(grupo_id, "🔍 Analisando canhoteira com IA...")
        
        # Simulação de caminho de imagem
        image_paths = ["/tmp/canhoteira.jpg"]
        canhoteiras_analisadas = analisar_multiplas_imagens(image_paths)
        
        entregas_db = get_entregas_por_carga(carga["id"])
        resultados = cruzar_dados(carga, canhoteiras_analisadas)
        
        for res in resultados:
            if res.status == "ENTREGUE":
                registrar_entrega_sucesso(carga["id"], res.nf_numero, res.__dict__)
            elif res.status == "PENDENTE":
                # Já está como pendente no banco, mas podemos atualizar se necessário
                pass

        # Mensagem de resumo da análise
        entregues = [r for r in resultados if r.status == "ENTREGUE"]
        pendentes = [r for r in resultados if r.status != "ENTREGUE"]
        
        msg = f"""📊 *ANÁLISE DA CANHOTEIRA*
━━━━━━━━━━━━━━━━━━━━
📸 Canhoteiras identificadas: *{len(canhoteiras_analisadas)}*
✅ Entregues: *{len(entregues)}*
❌ Pendentes: *{len(pendentes)}*

📦 *STATUS GERAL DA CARGA {carga['numero_carregamento']}:*
✅ Total Entregue: *{len([e for e in entregas_db if e['status'] == 'ENTREGUE']) + len(entregues)}/{len(entregas_db)}*
"""
        enviar_mensagem_grupo(grupo_id, msg)

        # Cobrança de motivos para pendentes
        for p in pendentes:
            cobranca = gerar_mensagem_cobranca(p, carga["numero_carregamento"])
            enviar_mensagem_grupo(grupo_id, cobranca)

    except Exception as e:
        print(f"Erro ao processar imagem: {e}")
        traceback.print_exc()
        enviar_mensagem_grupo(grupo_id, f"❌ Erro ao analisar a imagem: {str(e)}")

async def processar_resposta_motivo(data, grupo_id, texto):
    """Processa a resposta do motorista sobre o motivo da devolução."""
    carga = get_carga_ativa_por_grupo(grupo_id)
    if not carga: return

    aguardando = get_entregas_aguardando_motivo(carga["id"])
    if not aguardando: return

    motivo = interpretar_motivo(texto)
    entrega = aguardando[0]
    
    registrar_motivo_devolucao(carga["id"], entrega["nf_numero"], motivo)
    
    confirmacao = gerar_confirmacao_motivo(entrega["nf_numero"], motivo)
    enviar_mensagem_grupo(grupo_id, confirmacao)
    
    # Verificar se ainda há pendências
    ainda_aguardando = get_entregas_aguardando_motivo(carga["id"])
    if ainda_aguardando:
        prox = ainda_aguardando[0]
        enviar_mensagem_grupo(grupo_id, f"⚠️ Próxima pendência: NF-e *{prox['nf_numero']}* - {prox['cliente_nome']}\nInforme o motivo.")
    else:
        enviar_mensagem_grupo(grupo_id, "🏁 Todas as pendências da carga foram registradas!")

async def processar_comando(data, grupo_id, texto):
    sender = get_sender(data)
    texto = texto.lower().strip()
    
    if texto == "status":
        carga = get_carga_ativa_por_grupo(grupo_id)
        if not carga:
            enviar_mensagem_grupo(grupo_id, "📭 Nenhuma carga ativa neste grupo.")
            return True
        
        entregas = get_entregas_por_carga(carga["id"])
        entregues = [e for e in entregas if e["status"] == "ENTREGUE"]
        
        msg = f"""📊 *STATUS CARGA {carga['numero_carregamento']}*
👤 Motorista: {carga['motorista_nome']}
✅ Entregues: {len(entregues)}/{len(entregas)}
"""
        # Mostrar observações no status se houver pendências com obs
        pendentes_com_obs = [e for e in entregas if e["status"] != "ENTREGUE" and e["observacao"]]
        if pendentes_com_obs:
            msg += "\n⚠️ *OBSERVAÇÕES PENDENTES:*\n"
            for p in pendentes_com_obs:
                msg += f"• NF {p['nf_numero']}: {p['observacao']}\n"
                
        enviar_mensagem_grupo(grupo_id, msg)
        return True
    
    if texto == "finalizar" and is_admin(sender):
        carga = get_carga_ativa_por_grupo(grupo_id)
        if carga:
            finalizar_carga(carga["id"])
            enviar_mensagem_grupo(grupo_id, "✅ Carga finalizada com sucesso.")
        return True

    if texto == "ajuda":
        msg = """📖 *COMANDOS:*
• *status*: Ver progresso da carga
• *finalizar*: Encerrar carga (ADM)
• *ajuda*: Ver esta lista"""
        enviar_mensagem_grupo(grupo_id, msg)
        return True
        
    return False

@app.post("/webhook")
async def webhook(request: Request):
    try:
        data = await request.json()
    except:
        return JSONResponse({"status": "error"}, status_code=400)

    event = data.get("event")
    if event != "messages.upsert":
        return JSONResponse({"status": "ok"})

    if data.get("data", {}).get("key", {}).get("fromMe"):
        return JSONResponse({"status": "ok"})

    if not is_group_message(data):
        return JSONResponse({"status": "ok"})

    grupo_id = get_group_id(data)
    msg_type = get_message_type(data)
    texto = get_message_text(data)

    if msg_type == "text":
        if not await processar_comando(data, grupo_id, texto):
            await processar_resposta_motivo(data, grupo_id, texto)
    elif msg_type == "pdf":
        await processar_pdf_carga(data, grupo_id)
    elif msg_type == "image":
        await processar_imagem_canhoteira(data, grupo_id)

    return JSONResponse({"status": "ok"})

@app.get("/")
async def root():
    return {"status": "online", "service": "Bot Controle de Entregas"}
import sqlite3
import os
from datetime import datetime

DB_PATH = os.environ.get("DB_PATH", "data/entregas.db")

def get_connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = get_connection()
    
    # Tabela de motoristas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS motoristas (
            codigo TEXT PRIMARY KEY,
            nome TEXT NOT NULL
        )
    """)
    
    # Tabela de cargas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS cargas (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            numero_carregamento TEXT NOT NULL,
            motorista_codigo TEXT NOT NULL,
            motorista_nome TEXT,
            data_carga TEXT,
            grupo_whatsapp TEXT,
            status TEXT DEFAULT 'ATIVA',
            FOREIGN KEY (motorista_codigo) REFERENCES motoristas (codigo)
        )
    """)
    
    # Tabela de entregas
    conn.execute("""
        CREATE TABLE IF NOT EXISTS entregas (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            carga_id INTEGER NOT NULL,
            nf_numero TEXT NOT NULL,
            cliente_codigo TEXT,
            cliente_nome TEXT,
            cliente_fantasia TEXT,
            cnpj TEXT,
            endereco TEXT,
            telefone TEXT,
            total_volumes INTEGER,
            status TEXT DEFAULT 'PENDENTE',
            motivo_devolucao TEXT,
            data_recebimento TEXT,
            recebedor_nome TEXT,
            recebedor_documento TEXT,
            temperatura TEXT,
            observacao TEXT,
            FOREIGN KEY (carga_id) REFERENCES cargas (id)
        )
    """)
    
    # Tabela de logs de mensagens
    conn.execute("""
        CREATE TABLE IF NOT EXISTS mensagens (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            carga_id INTEGER,
            nf_numero TEXT,
            tipo TEXT,
            conteudo TEXT,
            remetente TEXT,
            grupo_id TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    conn.commit()
    conn.close()

def registrar_motorista(codigo, nome):
    conn = get_connection()
    conn.execute(
        "INSERT OR REPLACE INTO motoristas (codigo, nome) VALUES (?, ?)",
        (codigo, nome)
    )
    conn.commit()
    conn.close()

def listar_motoristas():
    conn = get_connection()
    rows = conn.execute("SELECT * FROM motoristas").fetchall()
    conn.close()
    return [dict(r) for r in rows]

def criar_carga(numero, motorista_cod, motorista_nome, data, grupo_id):
    registrar_motorista(motorista_cod, motorista_nome)
    conn = get_connection()
    cursor = conn.execute(
        "INSERT INTO cargas (numero_carregamento, motorista_codigo, motorista_nome, data_carga, grupo_whatsapp) VALUES (?, ?, ?, ?, ?)",
        (numero, motorista_cod, motorista_nome, data, grupo_id)
    )
    carga_id = cursor.lastrowid
    conn.commit()
    conn.close()
    return carga_id

def registrar_entrega(carga_id, nf_data):
    conn = get_connection()
    conn.execute("""
        INSERT INTO entregas (
            carga_id, nf_numero, cliente_codigo, cliente_nome, cliente_fantasia, 
            cnpj, endereco, telefone, total_volumes, status, observacao
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        carga_id, nf_data['numero'], nf_data.get('cliente_codigo'), nf_data.get('cliente_nome'),
        nf_data.get('cliente_fantasia'), nf_data.get('cnpj'), nf_data.get('endereco'),
        nf_data.get('telefone'), nf_data.get('total_volumes'), 'PENDENTE', nf_data.get('observacao')
    ))
    conn.commit()
    conn.close()

def get_carga_ativa_por_grupo(grupo_id):
    conn = get_connection()
    row = conn.execute(
        "SELECT * FROM cargas WHERE grupo_whatsapp = ? AND status = 'ATIVA' ORDER BY id DESC LIMIT 1",
        (grupo_id,)
    ).fetchone()
    conn.close()
    return dict(row) if row else None

def get_entregas_por_carga(carga_id):
    conn = get_connection()
    rows = conn.execute("SELECT * FROM entregas WHERE carga_id = ?", (carga_id,)).fetchall()
    conn.close()
    return [dict(r) for r in rows]

def registrar_motivo_devolucao(carga_id, nf_numero, motivo):
    conn = get_connection()
    conn.execute(
        "UPDATE entregas SET motivo_devolucao = ?, status = 'DEVOLVIDA' WHERE carga_id = ? AND nf_numero = ?",
        (motivo, carga_id, nf_numero)
    )
    conn.commit()
    conn.close()

def registrar_entrega_sucesso(carga_id, nf_numero, dados):
    conn = get_connection()
    conn.execute("""
        UPDATE entregas SET 
            status = 'ENTREGUE',
            data_recebimento = ?,
            recebedor_nome = ?,
            recebedor_documento = ?,
            temperatura = ?
        WHERE carga_id = ? AND nf_numero = ?
    """, (
        dados.get('data_recebimento'), dados.get('recebedor_nome'),
        dados.get('recebedor_documento'), dados.get('temperatura'),
        carga_id, nf_numero
    ))
    conn.commit()
    conn.close()

def finalizar_carga(carga_id):
    conn = get_connection()
    conn.execute("UPDATE cargas SET status = 'FINALIZADA' WHERE id = ?", (carga_id,))
    conn.commit()
    conn.close()

def registrar_mensagem(carga_id, nf_numero, tipo, conteudo, remetente, grupo_id):
    conn = get_connection()
    conn.execute("""
        INSERT INTO mensagens (carga_id, nf_numero, tipo, conteudo, remetente, grupo_id)
        VALUES (?, ?, ?, ?, ?, ?)
    """, (carga_id, nf_numero, tipo, conteudo, remetente, grupo_id))
    conn.commit()
    conn.close()

def get_entregas_aguardando_motivo(carga_id):
    conn = get_connection()
    rows = conn.execute(
        "SELECT * FROM entregas WHERE carga_id = ? AND status = 'PENDENTE' AND (motivo_devolucao IS NULL OR motivo_devolucao = '')",
        (carga_id,)
    ).fetchall()
    conn.close()
    return [dict(r) for r in rows]

# Funções para relatórios (DOC-20260302-WA0026)
def get_ranking_motoristas(data_inicio, data_fim):
    conn = get_connection()
    query = """
        SELECT m.nome, COUNT(e.id) as total, SUM(CASE WHEN e.status='ENTREGUE' THEN 1 ELSE 0 END) as entregues
        FROM motoristas m
        JOIN cargas c ON c.motorista_codigo = m.codigo
        JOIN entregas e ON e.carga_id = c.id
        WHERE c.data_carga BETWEEN ? AND ?
        GROUP BY m.nome
    """
    rows = conn.execute(query, (data_inicio, data_fim)).fetchall()
    conn.close()
    return [dict(r) for r in rows]

# Inicializar banco ao importar
os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
init_db()




