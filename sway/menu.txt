// Copyright 2017-2020 Nash Muhandes and Boondorl
// All rights reserved.
//
// Redistribution and use in source and binary forms, with or without
// modification, are permitted provided that the following conditions
// are met:
//
// 1. Redistributions of source code must retain the above copyright
//    notice, this list of conditions and the following disclaimer.
// 2. Redistributions in binary form must reproduce the above copyright
//    notice, this list of conditions and the following disclaimer in the
//    documentation and/or other materials provided with the distribution.
// 3. The name of the author may not be used to endorse or promote products
//    derived from this software without specific prior written permission.
//
// THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
// IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
// OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED.
// IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT, INDIRECT,
// INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
// NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
// DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
// THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
// (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF
// THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

class OptionMenuItemWSOption : OptionMenuItemOption
{
	String mTooltip;

	OptionMenuItemWSOption Init(String label, String tooltip, Name command, Name values, CVar graycheck = null, int center = 0)
	{
		mTooltip = tooltip;
		super.Init(label, command, values, graycheck, center);
		return self;
	}
}

class OptionMenuItemWSSlider : OptionMenuItemSlider
{
	String mTooltip;

	OptionMenuItemWSSlider Init(String label, String tooltip, Name command, double min, double max, double step, int showval = 1)
	{
		mTooltip = tooltip;
		super.Init(label, command, min, max, step, showval);
		return self;
	}
}

class WSOptionMenu : OptionMenu
{
	const START_TIME = 53;
	const END_TIME = 105;
	const SCROLL_SPEED = 5;
	const MAX_ASPECT = 16 / 9.;
	
	int mDefaultPosition;
	String mTooltip;
	
	private int startTimer;
	private int endTimer;
	private int scrollTimer;
	private int prevSelected;
	
	override void Init(Menu parent, OptionMenuDescriptor desc)
	{
		super.Init(parent, desc);
		
		mDefaultPosition = mDesc.mPosition;
		prevSelected = -1;
	}
	
	override void Drawer()
	{
		mToolTip = "";
		
		if (mDesc.mSelectedItem >= 0)
		{
			let item = mDesc.mItems[mDesc.mSelectedItem];
			if (item is "OptionMenuItemWSOption")
				mToolTip = StringTable.Localize(OptionMenuItemWSOption(item).mTooltip);
			else if (item is "OptionMenuItemWSSlider")
				mToolTip = StringTable.Localize(OptionMenuItemWSSlider(item).mTooltip);
		}
		
		Font of = OptionFont();
		int fHeight = of.GetHeight() * cleanYFac_1;
		int padding = fHeight<<1;
		
		if (prevSelected != mDesc.mSelectedItem)
		{
			startTimer = START_TIME;
			endTimer = 0;
			scrollTimer = 0;
		}
		
		if (mToolTip.Length() > 0)
		{
			int realWidth = Screen.GetWidth();
			int height = Screen.GetHeight();
			
			int width = realWidth;
			if (width / height > MAX_ASPECT)
				width = height * MAX_ASPECT;
			
			int textBoxWidth = width * 3/4.;
			int textBoxStart = width / 8 + (realWidth - width) / 2;
			
			int length = of.StringWidth(mToolTip) * cleanXFac_1;
			int xOfs = (realWidth - length) / 2;
			if (length > textBoxWidth)
			{
				xOfs = textBoxStart;
				if (startTimer <= 0)
				{
					xOfs -= SCROLL_SPEED * (endTimer <= 0 ? ++scrollTimer : scrollTimer);
					
					int end = xOfs + length;
					if (endTimer > 0 || end < textBoxStart + textBoxWidth)
					{
						xOfs += (textBoxStart + textBoxWidth - end);
						if (endTimer <= 0)
							endtimer = END_TIME;
					}
				}
				
				if (endTimer <= 0)
					textBoxWidth -= of.StringWidth("...") * cleanXFac_1;
			}
			
			int cx, cy, cw, ch;
			[cx, cy, cw, ch] = Screen.GetClipRect();
			Screen.SetClipRect(textBoxStart, padding, textBoxWidth, fHeight);
			
			Screen.DrawText(of, OptionMenuSettings.mFontColorValue,
							xOfs, padding,
							mToolTip,
							DTA_CleanNoMove_1, true);
							
			Screen.SetClipRect(cx, cy, cw, ch);
			
			if (length > textBoxWidth && endTimer <= 0)
			{
				Screen.DrawText(of, OptionMenuSettings.mFontColorValue,
								textBoxStart+textBoxWidth, padding,
								"...",
								DTA_CleanNoMove_1, true);
			}
		}
		
		if (startTimer > 0)
			--startTimer;
		
		if (endTimer > 0)
		{
			--endTimer;
			if (endTimer <= 0)
			{
				scrollTimer = 0;
				startTimer = START_TIME;
			}
		}
		
		int shift = -padding / cleanYFac_1;
		if (shift > mDefaultPosition)
			shift = mDefaultPosition;
			
		mDesc.mPosition = shift;
		prevSelected = mDesc.mSelectedItem;
		
		super.Drawer();
		
		mDesc.mPosition = mDefaultPosition;
	}
}
