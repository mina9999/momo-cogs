import asyncio
import logging
import random
from typing import Optional, TYPE_CHECKING, List, Union

import aiohttp
import discord
from io import BytesIO
from redbot.core import commands, Config
import UCube
import UCube.models
from UCube import UCubeClientAsync
from redbot.core.utils.chat_formatting import humanize_list, inline, pagify

logger = logging.getLogger('red.candycogs.UCube')


class ucube(commands.Cog):
    def __init__(self, bot, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.bot = bot

        self.config = Config.get_conf(self, identifier=7373854)
        self.config.register_global(token=None, seen=[])
        self.config.register_channel(channels={})

        self.ucube_client: Optional[UCubeClientAsync] = None
        self.session = aiohttp.ClientSession()
        self.ready = asyncio.Event()
        
        bot.loop.create_task(self.init())
        self._loop = bot.loop.create_task(self.run_loop())

    async def init(self):
        await self.bot.wait_until_red_ready()
        client_kwargs = {
            'username': 'your username from united-cube.com',  # ucube username
            'password': 'your password from united-cube.com',  # ucube password
            "verbose": True,  # Will print warning messages for links that have failed to connect or were not found.
            "web_session": self.session,  # Existing web session
        }
        self.ucube_client = UCubeClientAsync(**client_kwargs)
        start_kwargs = {
            "load_boards": True,
            "load_posts": False,
            "load_notices": False,
            "load_media": False,
            "load_from_artist": True,
            "load_to_artist": False,
            "load_talk": False,
            "load_comments": False,
            "follow_all_clubs": False
        }
        try:
            await self.ucube_client.start(**start_kwargs)
        except Exception:
            self.weverse_client = None
            raise
        finally:
            self.ready.set() 
            
    async def red_get_data_for_user(self, *, user_id):
        """Get a user's personal data."""
        data = "No data is stored for user with ID {}.\n".format(user_id)
        return {"user_data.txt": BytesIO(data.encode())}

    async def red_delete_data_for_user(self, *, requester, user_id):
        """Delete a user's personal data.

        No personal data is stored in this cog.
        """
        return
        
    def cog_unload(self):
        self._loop.cancel()
        self.bot.loop.create_task(self.session.close())
 
    async def run_loop(self):
        await self.bot.wait_until_red_ready()
        while True:
            try:
                await self.update_ucube()
            except asyncio.CancelledError:
                break
            except Exception:
                logger.exception("Error in loop")
            await asyncio.sleep(20)    
            
    @commands.group()
    async def ucube(self, ctx):
        """Subcommand for UCube related commands."""

    @ucube.command()
    @commands.is_owner()
    async def resetclient(self, ctx):
        """Reset the ucube client.  Do this when you've subscribed to new channels."""
        await self.init()
        await ctx.tick()

    @ucube.command(name="add")
    @commands.guild_only()
    @commands.has_guild_permissions(manage_messages=True)
    async def ucube_add(self, ctx, channel: Optional[discord.TextChannel], community_name, role: discord.Role = None):
        """Receive ucube updates from a specific ucube community.

        If the community is multiple words, surround the entire thing in quotes.
        """
        if channel is None:
            channel = ctx.channel
        if role is None:
            role_id = 0
        else:
            role_id = role.id

        community_name = community_name.lower()

        async with self.config.channel(channel).channels() as chans:
            if community_name in chans:
                if role_id != chans[community_name]['role_id']:
                    chans[community_name]['role_id'] = role_id
                    await ctx.send(f"Role updated for community `{community_name}`.")
                    return

            for community in self.ucube_client.clubs.values():
                if community.name.lower() == community_name:
                    chans[community_name] = {
                        'role_id': role_id,
                    }

                    await ctx.send(f"You will now receive ucube updates for {community_name}.")
                    return
            available = humanize_list([inline(com.name) for com in self.ucube_client.clubs.values()])  
            await ctx.send(f"I could not find {community_name}. Available choices are:\n" + available)

    @ucube.command(name="remove")
    @commands.guild_only()
    @commands.has_guild_permissions(manage_messages=True)
    async def ucube_remove(self, ctx, channel: Optional[discord.TextChannel], community_name):
        """Stop recieving ucube Updates from a specific ucube community in the current text channel.

        If the community is multiple words, surround the entire thing in quotes.
        """
        if channel is None:
            channel = ctx.channel

        async with self.config.channel(channel).channels() as chans:
            if community_name not in chans:
                await ctx.send("This community is not set up to notify the channel.")
                return
            del chans[community_name]
        await ctx.send(f"You will no longer receive ucube updates for {community_name}.")

    @ucube.command(name="list")
    @commands.guild_only()
    async def ucube_list(self, ctx, channel: discord.TextChannel = None):
        """Receive ucube updates from a specific ucube community.

        If the community is multiple words, surround the entire thing in quotes.
        """
        if channel is None:
            channel = ctx.channel

        chans = [inline(chan) for chan in await self.config.channel(channel).channels()]

        if not chans:
            await ctx.send("There are no communities set up to notify in this channel.")
            return

        await ctx.send(f"The communities set to notify in this channel are {humanize_list(chans)}.")

    async def update_ucube(self):
        """Process for checking for ucube updates and sending to discord channels."""         
        allchans = await self.config.all_channels()

        notifications = await self.ucube_client.check_new_notifications()
        await asyncio.sleep(2)
        for notification in notifications:
            post = self.ucube_client.get_post(notification.post_slug)
            club = self.ucube_client.get_club(notification.club_slug)
            if not post:
                continue            
            if post.slug in await self.config.seen():
                continue    
            async with self.config.seen() as seen:
                seen.append(post.slug)
            await asyncio.sleep(2)
            channels = [(c_id, data['channels'][club.name.lower()])
                        for c_id, data in allchans.items()
                        if club.name.lower() in data['channels']]        
            await asyncio.sleep(1)
            if not channels:
                continue   

            embed_title = f"New [{club.name}] {post.user.name} Notification!"
            translation = await self.translate(post.content)
            embed_description = (f"Content: **{post.content}**\n" +  
                                (f"\nTranslated Content: **{translation}**" if translation else ""))                                        
            result = []
            split = list(pagify(embed_description, shorten_by=24, page_length=6000))
            for sub in split:
                sub_split = list(pagify(sub, shorten_by=0, page_length=1024))
                desc = sub_split[0]
                if len(sub_split) > 1:
                    desc += sub_split[1]
                embed = discord.Embed(title=embed_title, description=None, color=discord.Color(random.randint(0x000000, 0xffffff)))
                embed.set_footer(text="💢Do .ucube for help💜", icon_url='https://cdn.discordapp.com/attachments/574296586742398997/870106439178403840/ezgif-2-441a54352e45.gif')
                embed.set_author(name="🧊Ucube", url="https://top.gg/bot/388331085060112397/",
                                 icon_url='https://cdn.discordapp.com/attachments/786160023000055839/890480488437915658/Cube_Entertainment_logo.png')
                embed.description = desc                     
            if len(sub_split) > 2:
                for x in sub_split[2:]:
                    embed.add_field(name='\u200B', value=x, inline=False)
            result.append(embed)
            embed = result[0]

            message_text = "\n".join(photo.path for photo in post.images)

            if not embed:
                continue
  
            for channel_id, data in channels:
                await self.send_ucube_to_channel(channel_id, data, message_text, embed, club.name)

    async def send_ucube_to_channel(self, channel_id, channel_data, message_text, embed, club_name):
        role_id = channel_data['role_id']        
        if embed:
            channel = self.bot.get_channel(channel_id)
            try:  
                if channel.permissions_for(channel.guild.me).manage_webhooks:
                    webhook = None
                    for hook in await channel.webhooks():
                        if hook.name == channel.guild.me.name:
                            webhook = hook
                    if webhook is None:
                        webhook = await channel.create_webhook(name=channel.guild.me.name) 
                    avatar = "https://cdn.discordapp.com/attachments/786160023000055839/890480488437915658/Cube_Entertainment_logo.png"
                    username = f"{club_name}"
                    mention_role = f"<@&{role_id}>" if role_id else None                
                    message = await webhook.send(mention_role, username=username, avatar_url=avatar, embed=embed, allowed_mentions=discord.AllowedMentions(roles=True), wait=True)
                    await message.add_reaction("<a:eatinglove:890166530246082601>")
                    await asyncio.sleep(5)
                    if message_text:
                        for url in message_text.split("\n"):
                            msg3 = await webhook.send(url, username=username, avatar_url=avatar, wait=True)
                            await msg3.add_reaction("<a:loveweverse:890143261614825502>")                    
            except Exception:
                pass
        
    async def translate(self, text: str) -> Optional[str]:
        if self.bot.get_cog("Papago"):
            try:
                return await self.bot.get_cog("Papago").translate('ko', 'en', text)
            except ValueError:
                pass
            except Exception:
                logger.exception("Exception: ")
