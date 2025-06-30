import discord
from discord.ext import commands

# 🔧 CONFIGURAÇÕES
TOKEN = 'MTM4OTAzMDY4MzQxODAzODI4Mw.GwuFbq.PgEC9ZvPcbhxwVNR5TkXXHzFFuqIJdKU4I-N58'  # Seu token
ID_CANAL_ANALISE = 1351225811964924055  # Canal dos pedidos
ID_CARGO_WHITELIST = 1300956630359343215  # Cargo que será dado

intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix="!", intents=intents)


# ✅ Quando o bot liga
@bot.event
async def on_ready():
    print(f"✅ Bot online como {bot.user}")


# ✅ Comando para enviar o botão da whitelist
@bot.command()
async def setup(ctx):
    embed = discord.Embed(
        title="🚀 Pedido de Whitelist",
        description="Clique no botão abaixo para solicitar sua whitelist.",
        color=discord.Color.blue()
    )
    view = WhitelistButton()
    await ctx.send(embed=embed, view=view)


# ✅ Classe do botão que abre o formulário
class WhitelistButton(discord.ui.View):
    @discord.ui.button(label="Fazer Whitelist", style=discord.ButtonStyle.green)
    async def whitelist_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_modal(WhitelistModal())


# ✅ Formulário para o usuário preencher
class WhitelistModal(discord.ui.Modal, title="Formulário de Whitelist"):
    idjogo = discord.ui.TextInput(label="Seu ID no jogo", placeholder="Ex: 190", required=True)
    nome = discord.ui.TextInput(label="Seu nome no jogo", placeholder="Ex: Souza Argentino", required=True)

    async def on_submit(self, interaction: discord.Interaction):
        canal = bot.get_channel(ID_CANAL_ANALISE)

        embed = discord.Embed(
            title="📝 Novo Pedido de Whitelist",
            color=discord.Color.blurple()
        )
        embed.add_field(name="👤 Usuário", value=interaction.user.mention, inline=False)
        embed.add_field(name="🆔 ID", value=self.idjogo.value, inline=False)
        embed.add_field(name="📛 Nome", value=self.nome.value, inline=False)

        view = AprovarOuRecusarView(user_id=interaction.user.id)

        await canal.send(embed=embed, view=view)
        await interaction.response.send_message(
            "✅ Seu pedido foi enviado para análise. Aguarde a resposta de um administrador.",
            ephemeral=True
        )


# ✅ Botões para Aprovar ou Recusar
class AprovarOuRecusarView(discord.ui.View):
    def __init__(self, user_id):
        super().__init__(timeout=None)
        self.user_id = user_id

    @discord.ui.button(label="✅ Aprovar", style=discord.ButtonStyle.success)
    async def aprovar(self, interaction: discord.Interaction, button: discord.ui.Button):
        guild = interaction.guild
        membro = guild.get_member(self.user_id)

        if membro is None:
            await interaction.response.send_message("❌ Usuário não encontrado no servidor.", ephemeral=True)
            return

        cargo = guild.get_role(ID_CARGO_WHITELIST)
        await membro.add_roles(cargo)

        # 🔥 Renomear o nick do usuário com Nome | ID
        nome = None
        idjogo = None

        for field in interaction.message.embeds[0].fields:
            if field.name == "🆔 ID":
                idjogo = field.value
            if field.name == "📛 Nome":
                nome = field.value

        if nome and idjogo:
            novo_nick = f"{nome} | {idjogo}"
            try:
                await membro.edit(nick=novo_nick)
            except:
                await interaction.channel.send(f"⚠️ Não consegui alterar o nick de {membro.mention}. Verifique as permissões do bot.")

        await interaction.response.send_message(f"✅ {membro.mention} foi aprovado na whitelist, recebeu o cargo e teve o nome alterado.", ephemeral=False)
        await membro.send("✅ Você foi **aprovado na whitelist!** Seja bem-vindo!")

        self.disable_all_items()
        await interaction.message.edit(view=self)

    @discord.ui.button(label="❌ Recusar", style=discord.ButtonStyle.danger)
    async def recusar(self, interaction: discord.Interaction, button: discord.ui.Button):
        guild = interaction.guild
        membro = guild.get_member(self.user_id)

        if membro:
            try:
                await membro.send("❌ Seu pedido de whitelist foi recusado.")
            except:
                pass  # Caso o usuário esteja com o privado fechado

        await interaction.response.send_message(f"❌ Pedido recusado.", ephemeral=False)

        self.disable_all_items()
        await interaction.message.edit(view=self)


bot.run(TOKEN)
