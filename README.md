import discord
from discord.ext import commands
import asyncio
import datetime

# Configurazione del bot
TOKEN = ""
PREFIX = "!"
INTENTS = discord.Intents.default()
INTENTS.members = True
INTENTS.message_content = True

# Creazione del bot
bot = commands.Bot(command_prefix=PREFIX, intents=INTENTS)

# Evento di avvio
@bot.event
async def on_ready():
    print(f'Bot avviato come {bot.user.name}')
    print(f'ID: {bot.user.id}')
    print('------')
    await bot.change_presence(activity=discord.Game(name="Moderazione Server"))

# Comandi di moderazione

@bot.command(name='kick', help='Espelle un membro dal server')
@commands.has_permissions(kick_members=True)
async def kick(ctx, member: discord.Member, *, reason=None):
    await member.kick(reason=reason)
    await ctx.send(f'{member.mention} è stato espulso. Motivo: {reason}')

@bot.command(name='ban', help='Banna un membro dal server')
@commands.has_permissions(ban_members=True)
async def ban(ctx, member: discord.Member, *, reason=None):
    await member.ban(reason=reason)
    await ctx.send(f'{member.mention} è stato bannato. Motivo: {reason}')

@bot.command(name='unban', help='Rimuove il ban a un utente')
@commands.has_permissions(ban_members=True)
async def unban(ctx, *, member):
    banned_users = await ctx.guild.bans()
    member_name, member_discriminator = member.split('#')

    for ban_entry in banned_users:
        user = ban_entry.user

        if (user.name, user.discriminator) == (member_name, member_discriminator):
            await ctx.guild.unban(user)
            await ctx.send(f'{user.mention} è stato sbannato.')
            return

@bot.command(name='mute', help='Muta un membro')
@commands.has_permissions(manage_roles=True)
async def mute(ctx, member: discord.Member, *, reason=None):
    muted_role = discord.utils.get(ctx.guild.roles, name="Muted")

    if not muted_role:
        # Crea il ruolo Muted se non esiste
        muted_role = await ctx.guild.create_role(name="Muted")
        
        # Disabilita i permessi di parola
        for channel in ctx.guild.channels:
            await channel.set_permissions(muted_role, speak=False, send_messages=False)

    await member.add_roles(muted_role, reason=reason)
    await ctx.send(f'{member.mention} è stato mutato. Motivo: {reason}')

@bot.command(name='unmute', help='Rimuove il mute a un membro')
@commands.has_permissions(manage_roles=True)
async def unmute(ctx, member: discord.Member):
    muted_role = discord.utils.get(ctx.guild.roles, name="Muted")

    if muted_role in member.roles:
        await member.remove_roles(muted_role)
        await ctx.send(f'{member.mention} è stato smutato.')
    else:
        await ctx.send(f'{member.mention} non è mutato.')

@bot.command(name='clear', help='Cancella un numero specificato di messaggi')
@commands.has_permissions(manage_messages=True)
async def clear(ctx, amount: int):
    await ctx.channel.purge(limit=amount + 1)
    msg = await ctx.send(f'{amount} messaggi cancellati.')
    await asyncio.sleep(3)
    await msg.delete()

@bot.command(name='warn', help='Avvisa un membro')
@commands.has_permissions(kick_members=True)
async def warn(ctx, member: discord.Member, *, reason=None):
    await ctx.send(f'{member.mention} ha ricevuto un avvertimento. Motivo: {reason}')

# Gestione degli errori
@kick.error
@ban.error
@mute.error
@clear.error
async def moderation_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("Non hai i permessi necessari per eseguire questo comando.")
    elif isinstance(error, commands.BadArgument):
        await ctx.send("Membro non trovato o formato non valido.")
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send("Per favore specifica un membro.")

# Avvio del bot
if __name__ == "__main__":
    bot.run(TOKEN)
