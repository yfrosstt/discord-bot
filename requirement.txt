import discord
from discord.ext import commands
import json
import os
from datetime import timedelta

# ─────────────────────────────────────────────
#  CONFIG
# ─────────────────────────────────────────────
PREFIX = "&"
BLACKLIST_FILE = "blacklist.json"

intents = discord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(command_prefix=PREFIX, intents=intents)

# ─────────────────────────────────────────────
#  BLACKLIST HELPERS
# ─────────────────────────────────────────────
def load_blacklist() -> list[int]:
    if os.path.exists(BLACKLIST_FILE):
        with open(BLACKLIST_FILE, "r") as f:
            return json.load(f)
    return []

def save_blacklist(data: list[int]) -> None:
    with open(BLACKLIST_FILE, "w") as f:
        json.dump(data, f, indent=2)

# ─────────────────────────────────────────────
#  HELPER — résoudre un ID en Member ou User
# ─────────────────────────────────────────────
async def resolve_member(ctx: commands.Context, user_id: int) -> discord.Member | None:
    member = ctx.guild.get_member(user_id)
    if member is None:
        try:
            member = await ctx.guild.fetch_member(user_id)
        except (discord.NotFound, discord.HTTPException):
            member = None
    return member

# ─────────────────────────────────────────────
#  BLACKLIST CHECK  (appliqué à chaque commande)
# ─────────────────────────────────────────────
@bot.check
async def global_blacklist_check(ctx: commands.Context) -> bool:
    bl = load_blacklist()
    if ctx.author.id in bl:
        await ctx.send(f"🚫 {ctx.author.mention}, tu es **blacklisté** et ne peux pas utiliser les commandes.")
        return False
    return True

# ─────────────────────────────────────────────
#  EVENTS
# ─────────────────────────────────────────────
@bot.event
async def on_ready():
    print(f"✅ Connecté en tant que {bot.user} (ID: {bot.user.id})")
    print(f"   Préfixe : {PREFIX}")

@bot.event
async def on_command_error(ctx: commands.Context, error):
    if isinstance(error, commands.CheckFailure):
        return
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("❌ Tu n'as pas la permission d'utiliser cette commande.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"❌ Argument manquant : `{error.param.name}`")
    else:
        await ctx.send(f"❌ Erreur : `{error}`")
        raise error

# ─────────────────────────────────────────────
#  &mute <ID> [durée_en_minutes]
# ─────────────────────────────────────────────
@bot.command(name="mute")
@commands.has_permissions(moderate_members=True)
async def mute(ctx: commands.Context, user_id: int, duration: int = 10):
    """&mute <ID> [durée_en_minutes]"""
    member = await resolve_member(ctx, user_id)
    if member is None:
        await ctx.send(f"❌ Aucun membre trouvé avec l'ID `{user_id}`.")
        return

    if member.top_role >= ctx.author.top_role and ctx.author != ctx.guild.owner:
        await ctx.send("❌ Tu ne peux pas muter quelqu'un avec un rôle supérieur ou égal au tien.")
        return

    duration = max(1, min(duration, 40320))
    until = discord.utils.utcnow() + timedelta(minutes=duration)

    await member.timeout(until)

    embed = discord.Embed(title="🔇 Membre muté", color=discord.Color.orange())
    embed.add_field(name="Membre", value=f"{member} (`{member.id}`)", inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    embed.add_field(name="Durée", value=f"{duration} minute(s)", inline=True)
    embed.set_footer(text=f"Fin : {until.strftime('%d/%m/%Y %H:%M')} UTC")
    await ctx.send(embed=embed)

# ─────────────────────────────────────────────
#  &unmute <ID>
# ─────────────────────────────────────────────
@bot.command(name="unmute")
@commands.has_permissions(moderate_members=True)
async def unmute(ctx: commands.Context, user_id: int):
    """&unmute <ID>"""
    member = await resolve_member(ctx, user_id)
    if member is None:
        await ctx.send(f"❌ Aucun membre trouvé avec l'ID `{user_id}`.")
        return

    if not member.is_timed_out():
        await ctx.send(f"ℹ️ {member} n'est pas muté.")
        return

    await member.timeout(None)

    embed = discord.Embed(title="🔊 Membre démuté", color=discord.Color.green())
    embed.add_field(name="Membre", value=f"{member} (`{member.id}`)", inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    await ctx.send(embed=embed)

# ─────────────────────────────────────────────
#  &bl <ID>          → blackliste
#  &unbl <ID>        → retire
#  &bl list          → affiche la liste
# ─────────────────────────────────────────────
@bot.command(name="bl")
@commands.has_permissions(administrator=True)
async def blacklist_add(ctx: commands.Context, user_id: int):
    """&bl <ID>  — Blackliste un membre"""
    bl = load_blacklist()
    if user_id in bl:
        await ctx.send(f"ℹ️ L'ID `{user_id}` est déjà blacklisté.")
        return

    bl.append(user_id)
    save_blacklist(bl)

    member = await resolve_member(ctx, user_id)
    display = f"{member} (`{member.id}`)" if member else f"ID `{user_id}`"

    embed = discord.Embed(title="⛔ Blacklist — Ajout", color=discord.Color.red())
    embed.add_field(name="Membre", value=display, inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    await ctx.send(embed=embed)

@bot.command(name="unbl")
@commands.has_permissions(administrator=True)
async def blacklist_remove(ctx: commands.Context, user_id: int):
    """&unbl <ID>  — Retire un membre de la blacklist"""
    bl = load_blacklist()
    if user_id not in bl:
        await ctx.send(f"ℹ️ L'ID `{user_id}` n'est pas blacklisté.")
        return

    bl.remove(user_id)
    save_blacklist(bl)

    member = await resolve_member(ctx, user_id)
    display = f"{member} (`{member.id}`)" if member else f"ID `{user_id}`"

    embed = discord.Embed(title="✅ Blacklist — Retrait", color=discord.Color.green())
    embed.add_field(name="Membre", value=display, inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    await ctx.send(embed=embed)

@bot.command(name="bllist")
@commands.has_permissions(administrator=True)
async def blacklist_list(ctx: commands.Context):
    """&bllist  — Affiche la blacklist"""
    bl = load_blacklist()
    if not bl:
        await ctx.send("📋 La blacklist est vide.")
        return

    lines = []
    for uid in bl:
        m = await resolve_member(ctx, uid)
        lines.append(f"• {m} (`{uid}`)" if m else f"• ID `{uid}`")

    embed = discord.Embed(
        title="📋 Blacklist",
        description="\n".join(lines),
        color=discord.Color.dark_red()
    )
    embed.set_footer(text=f"{len(bl)} membre(s) blacklisté(s)")
    await ctx.send(embed=embed)

# ─────────────────────────────────────────────
#  &ban <ID> [raison]
# ─────────────────────────────────────────────
@bot.command(name="ban")
@commands.has_permissions(ban_members=True)
async def ban(ctx: commands.Context, user_id: int, *, reason: str = "Aucune raison précisée"):
    """&ban <ID> [raison]"""
    member = await resolve_member(ctx, user_id)

    if member:
        if member.top_role >= ctx.author.top_role and ctx.author != ctx.guild.owner:
            await ctx.send("❌ Tu ne peux pas bannir quelqu'un avec un rôle supérieur ou égal au tien.")
            return
        try:
            await member.send(
                f"🔨 Tu as été **banni** du serveur **{ctx.guild.name}**.\n"
                f"Raison : {reason}"
            )
        except discord.Forbidden:
            pass
        await member.ban(delete_message_days=0, reason=reason)
        display = f"{member} (`{member.id}`)"
        avatar = member.display_avatar.url
    else:
        # Bannissement par ID même si le membre n'est plus sur le serveur
        try:
            user = await bot.fetch_user(user_id)
            await ctx.guild.ban(user, delete_message_days=0, reason=reason)
            display = f"{user} (`{user.id}`)"
            avatar = user.display_avatar.url
        except discord.NotFound:
            await ctx.send(f"❌ Aucun utilisateur trouvé avec l'ID `{user_id}`.")
            return

    embed = discord.Embed(title="🔨 Membre banni", color=discord.Color.dark_red())
    embed.add_field(name="Membre", value=display, inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    embed.add_field(name="Raison", value=reason, inline=False)
    embed.set_thumbnail(url=avatar)
    await ctx.send(embed=embed)

# ─────────────────────────────────────────────
#  &unban <ID>
# ─────────────────────────────────────────────
@bot.command(name="unban")
@commands.has_permissions(ban_members=True)
async def unban(ctx: commands.Context, user_id: int):
    """&unban <ID>"""
    try:
        user = await bot.fetch_user(user_id)
    except discord.NotFound:
        await ctx.send(f"❌ Aucun utilisateur trouvé avec l'ID `{user_id}`.")
        return

    try:
        await ctx.guild.unban(user)
    except discord.NotFound:
        await ctx.send(f"ℹ️ `{user}` n'est pas banni de ce serveur.")
        return

    embed = discord.Embed(title="✅ Membre débanni", color=discord.Color.green())
    embed.add_field(name="Membre", value=f"{user} (`{user.id}`)", inline=True)
    embed.add_field(name="Par", value=ctx.author.mention, inline=True)
    embed.set_thumbnail(url=user.display_avatar.url)
    await ctx.send(embed=embed)

# ─────────────────────────────────────────────
#  LANCEMENT
# ─────────────────────────────────────────────
if __name__ == "__main__":
    TOKEN = os.getenv("DISCORD_TOKEN")
    if not TOKEN:
        raise RuntimeError(
            "Token manquant ! Définis la variable d'environnement DISCORD_TOKEN.\n"
            "Exemple : export DISCORD_TOKEN='ton_token_ici'"
        )
    bot.run(TOKEN)
